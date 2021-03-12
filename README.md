OpenAPI Governor
================

Enforce rules against OpenAPI (fka Swagger) specifications.

## Example

Say you have added a new, required, parameter in the latest version of your OpenAPI specification.

When you run Governor, passing the path of the previous and latest versions of your OpenAPI specifications, the `required-parameters-added` rule will fail with the following message:

    $ governor -s ./examples/petstore_v2.yaml -p ./examples/petstore_v1.yaml -r ./examples/ruleset.yaml
    
    Required parameter 'category' in GET /pets: new in latest version

This check works for the following scenarios:

- an existing parameter is made required (i.e. newly mandatory)
- a new parameter is added, and marked as required
- a new path or operation is added containing required parameters

## Rules

In addition to the `required-parameters-added` rule, you can set a variety of other rules for your OpenAPI specifications. For example, you can require certain fields are populated, present, or absent.

```
# ruleset.yaml
rules:
  # check if there are newly required parameters
  - required-parameters-added
  
  # ensure the title matches the expected value
  - value-at-path:
      path: $.info.title
      operator: EqualTo
      value: Swagger Petstore

  # ensure the location is not blank
  - value-at-path:
      path: $.info.location
      operator: Exists
```

The `value-at-path` rule supports the following operators:

- Exists
- NotExists
- Blank
- NotBlank
- EqualTo
- NotEqualTo

## Quick start

The `examples` directory contains sample OpenAPI specifications and rules.

If you have Docker installed, use the convenience script that wraps the Docker image build, then
starts a container passing through arguments.

Prerequisites:
- JDK 8 or newer
- Docker

Usage:

    ./governor.sh [args]

> See below for list of valid arguments.

Example:

    $ ./governor.sh -s ./examples/petstore_v2.yaml -p ./examples/petstore_v1.yaml -r ./examples/ruleset.yaml
    
    WARN  RuleEnforcer - Some rules failed.
    
    Failed (2):
    ❌   required-parameters-added: Required parameter 'category' in GET /pets: new in latest version
    ❌   value-at-path: mismatched value at: $.info.location - expected exists, actual: null
    
    Passed (1):
    ✅   value-at-path: value at: $.info.title == Swagger Petstore

## Running Governor

Governor images are pushed to [Docker Hub](https://hub.docker.com/r/outofcoffee/governor).

> See the `examples` directory in this repository for sample files.

Let's start with the following file structure:

```
specs/
 ∟ petstore_v1.yaml
 ∟ ruleset.yaml
```

In this example, `petstore_v1.yaml` is your OpenAPI specification. The `ruleset.yaml` file lists the rules you want to enforce against your specification.

Here's an example ruleset:

```
# ruleset.yaml
rules:
  # ensure the title matches the expected value
  - value-at-path:
      path: $.info.title
      operator: EqualTo
      value: Swagger Petstore

  # ensure the location is not blank
  - value-at-path:
      path: $.info.location
      operator: Exists
```

Run Governor as follows:

    docker run --rm -it -v $PWD/specs:/app/specs outofcoffee/governor \
                -s ./specs/petstore_v1.yaml -r ./specs/ruleset.yaml

> Note that this uses the bind-mount mechanism in Docker. Therefore the `./specs` prefix for the files, refers to the path `/app/specs` within the container filesystem.

### Comparing between versions

Some rules allow you to check for differences between specification versions. For example, you may have modified your specification to introduce a new mandatory parameter. Knowing this might be important to help you avoid backwards compatibility issues as you roll out your new API version.

Let's add a second version of our OpenAPI specification, `petstore_v2.yaml`. For the purposes of this example, let's add a new, required, parameter to v2 of the specification. This results in the following file structure:

```
specs/
 ∟ petstore_v1.yaml
 ∟ petstore_v2.yaml
 ∟ ruleset.yaml
```

Now add the following rule to our ruleset file:

```
  # check if there are newly required parameters
  - required-parameters-added
``` 

Run Governor as follows - note the use of the `-p` (previous version) flag:

    docker run --rm -it -v $PWD/specs:/app/specs outofcoffee/governor \
                -s ./specs/petstore_v2.yaml -p ./specs/petstore_v1.yaml -r ./specs/ruleset.yaml

As there is a new mandatory ('required') parameter between v1 and v2 of the specification, the `required-parameters-added` rule will pick this up.

### Usage

```
Usage: governor options_list
Options: 
    --currentSpecFile, -s -> OpenAPI specification file (always required) { String }
    --previousSpecFile, -p -> Previous OpenAPI specification file { String }
    --rulesFile, -r -> Rules file (always required) { String }
    --nonZeroExitCodeOnFailure, -z -> Return a non-zero exit code if rule evaluation fails
    --help, -h -> Usage info
```

### Exit code

When Governor runs, it produces output indicating which rules pass and fail. By default, it will exit with status 0. If you'd like the exit code to reflect whether all rules were evaluated successfully, pass the `-z` flag. In this case, if all rules pass, the exit code will be 0, otherwise it will be 1.

This can be useful when scripting your CI/CD pipeline to cause a build or deployment to fail if rules are not satisfied.

## Building

If you don't want to use the prebuilt Docker image, you can build Governor yourself.

### With Docker

This method builds a Docker image, named `outofcoffee/governor:latest`. You can then bind-mount a directory containing your specifications, use it as a base image etc.

Build:

    ./gradlew dockerBuildImage

Test:

    docker run --rm -it -v $PWD/examples:/app/examples outofcoffee/governor:latest \
            -s ./examples/petstore_v2.yaml -p ./examples/petstore_v1.yaml -r ./examples/ruleset.yaml

### Without Docker

This method builds an application distributable, with entrypoint scripts for *NIX and Windows, under the `build/install/openapi-governor/bin` directory.

Build:

    ./gradlew installDist

Test:

    ./build/install/openapi-governor/bin/openapi-governor -s ./examples/petstore_v2.yaml -p ./examples/petstore_v1.yaml -r ./examples/ruleset.yaml
