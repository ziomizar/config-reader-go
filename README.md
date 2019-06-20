# Platform.sh Config Reader (Go)

This library provides a streamlined and easy to use way to interact the a Platform.sh environment.  It defines structs for Routes and Relationships and offers utility methods to access them more cleanly than reading the raw environment variables yourself.

This library is best installed using Go modules in Go 1.11 and later.

## Install

Add a dependency on `github.com/platformsh/config-reader-go` to your application. We recommend giving it an explicit import name.

## Usage Example

Example:

```go
package main

import (
	_ "github.com/go-sql-driver/mysql"
	psh "github.com/platformsh/config-reader-go"
	"net/http"
)

func main() {

	p, err := psh.NewRuntimeConfig()
	if err != nil {
		panic("Not in a Platform.sh Environment.")
	}

	dbString, err := p.FormattedCredentials("database", "sqldsn")

  // Use the db connection here, using type assertion.

	// Set up an extremely simple web server response.
	http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
		// ...
	})

    // Note the Port value used here.
	http.ListenAndServe(":"+p.Port(), nil)
}
```

## API Reference

### Create a config object

There are two separate constructor functions depending on whether you intend to be in a build environment or runtime environment.

```go
// In a build hook, run:
buildConfig, err := psh.NewBuildConfig()
if err != nil {
    panic("Not in a Platform.sh Environment.")
}
```

`buildConfig` is now a `psh.BuildConfig` struct that provides access to the Platform.sh build environment context.  If `err` is not `nil` it means the library is not running on Platform.sh, so other commands would not run.

```go
// At runtime, run:
runtimeConfig, err := psh.NewRuntimeConfig()
if err != nil {
    panic("Not in a Platform.sh Environment.")
}
```

`runtimeConfig` is now a `psh.RuntimeConfig` struct that provides access to the Platform.sh runtime environment context.  That includes everything available in the Build context as well as information only meaningful at runtime.

### Inspect the environment

The following methods return `true` or `false` to help determine in what context the code is running:

```go
runtimeConfig.OnEnterprise()

runtimeConfig.OnProduction()
```

### Read environment variables

The following methods return the corresponding environment variable value.  See the [Platform.sh documentation](https://docs.platform.sh/development/variables.html) for a description of each.

The following are available both in Build and at Runtime:

```go
buildConfig.ApplicationName()

buildConfig.AppDir()

buildConfig.Project()

buildConfig.TreeId()

buildConfig.ProjectEntropy()
```

The following are available only on a `RuntimeConfig` struct:

```go
runtimeConfig.Branch()

runtimeConfig.DocumentRoot()

runtimeConfig.SmtpHost()

runtimeConfig.Environment()

runtimeConfig.Socket()

runtimeConfig.Port()
```

### Reading service credentials

[Platform.sh services](https://docs.platform.sh/configuration/services.html) are defined in a `services.yaml` file, and exposed to an application by listing a `relationship` to that service in the application's `.platform.app.yaml` file.  User, password, host, etc. information is then exposed to the running application in the `PLATFORM_RELATIONSHIPS` environment variable, which is a base64-encoded JSON string.  The following method allows easier access to credential information than decoding the environment variable yourself.

```go
if creds, ok := runtimeConfig.Credentials("database"); ok {
	// ...
}
```

The return value of `Credentials()` is a `Credential` struct, which includes the appropriate user, password, host, database name, and other pertinent information.  See the [Service documentation](https://docs.platform.sh/configuration/services.html) for your service for the exact structure and meaning of each property.  In most cases that information can be passed directly to whatever other client library is being used to connect to the service.

If `ok` is false it means the specified relationship was not defined so no credentials are available.

## Formatting service credentials

In some cases the library being used to connect to a service wants its credentials formatted in a specific way; it could be a DSN string of some sort or it needs certain values concatenated to the database name, etc.  For those cases you can use "Credential Formatters".  A Credential Formatter is a function that takes a `Credential` object and returns any type, since the library may want different types.  They must conform to the `CredentialFormatter` type defined in this package.

Credential Formatters can be registered on the configuration object, and one is included out of the box.  That allows 3rd party libraries to ship their own formatters that can be easily integrated into the `Config` object to allow easier use.

```go
func formatMyService(creds Credential) interface{} {
	return "some string based on creds";
}

// Call this in setup.
runtimeConfig.RegisterFormatter("my_service", formatMyService)


// Then call this method to get the formatted version

formatted, err := runtimeConfig.FormattedCredentials("database", "my_service")
```

The first parameter is the name of a relationship defined in `.platform.app.yaml`.  The second is a formatter that was previously registered with `RegisterFormatter()`.  `err` will be non-`nil` if either relationship or formatter name is missing.  The type of `formatted` will depend on the formatter function.  If `err` is `nil` then it is safe to then pass to the appropriate client library.

### Reading Platform.sh variables

Platform.sh allows you to define arbitrary variables that may be available at build time, runtime, or both.  They are stored in the `PLATFORM_VARIABLES` environment variable, which is a base64-encoded JSON string.  

The following two methods allow access to those values from your code without having to bother decoding the values yourself:

```go
runtimeConfig.Variables()
```

This method returns a `map[string]string` of all variables defined.  Usually this method is not necessary and `config.Variable()` is preferred.

```go
runtimeConfig.Variable("foo", "default")
```

This method looks for the "foo" variable.  If found, it is returned.  If not, the second parameter is returned as a default.

Note that both methods are available on both Build and Runtime, although different values may be defined and avaialble for use.

### Reading Routes

[Routes](https://docs.platform.sh/configuration/routes.html) on Platform.sh define how a project will handle incoming requests; that primarily means what application container will serve the request, but it also includes cache configuration, TLS settings, etc.  Routes may also have an optional ID, which is the preferred way to access them.

```go
runtimeConfig.Route("main")
```

The `Route()` method takes a single string for the route ID ("main" in this case) and returns the corresponding `route` struct.  Its second return is a boolean indicating if the route was found.  It is best used like so:

```go
if route, ok := runtimeConfig.Route("main"); ok {
	// The route was found, so do stuff with `route`
}
```

To access all routes, or to search for a route that has no ID, the `Routes()` method returns a `map[string]Route` of URLs to `Route` objects.  That mirrors the structure of the `PLATFORM_ROUTES` environment variable.

```go
routes := runtimeConfig.Routes()
```
