![UI5 logo](https://github.com/SAP/ui5-tooling/blob/master/docs/images/UI5_logo_wide.png)

# ui5-project
> Modules for building a UI5 projects dependency tree, including configuration  
> Part of the [UI5 Build and Development Tooling](https://github.com/SAP/ui5-tooling)

[![Travis CI Build Status](https://travis-ci.org/SAP/ui5-project.svg?branch=master)](https://travis-ci.org/SAP/ui5-project)

**This is a Pre-Alpha release!**  
**The UI5 Build and Development Tooling described here is not intended for productive use yet. Breaking changes are to be expected.**

## Normalizer
The purpose of the normalizer is to collect dependency information and to enrich it with project configuration ([generateProjectTree](https://github.com/pages/SAP/ui5-tooling/module-normalizer_normalizer.html#~generateProjectTree)).

[Translators](#translators) are used to collect dependency information. The [Project Preprocessor](#project-preprocessor) enriches this dependency information with project configuration. Typically from a `ui5.yaml` file. A development server and build process can use this information to locate project- and dependency resources.

To only retrieve the project dependency graph, please use ([generateDependencyTree](https://github.com/pages/SAP/ui5-tooling/module-normalizer_normalizer.html#~generateDependencyTree)).

## Translators
Translators collect recursively all dependencies on a package manager specific layer and return information about them in a well-defined tree-structure.

### Tree structure returned by a translator
The following dependency tree is the expected input structure of the [Project Preprocessor](#project-preprocessor).

````json
{
    "id": "projectA",
    "version": "1.0.0",
    "path": "/absolute/path/to/projectA",
    "dependencies": [
        {
            "id": "projectB",
            "version": "1.0.0",
            "path": "/path/to/projectB",
            "dependencies": [
                {
                    "id": "projectD",
                    "path": "/path/to/different/projectD"
                }
            ]
        },
        {
            "id": "projectD",
            "version": "1.0.0",
            "path": "/path/to/projectD"
        },
        {
            "id": "myStaticServerTool",
            "version": "1.0.0",
            "path": "/path/to/some/dependency"
        }
    ]
}
````

### npm Translator
The npm translator is currently the default translator and looks for dependencies defined in the `package.json` file of a certain project. `dependencies`, `devDepedencies` and [napa dependencies](https://github.com/shama/napa) (Git repositories which don't have a `package.json` file) are located via the Node.js module resolution logic.

### Static Translator
*This translator is currently intended for testing purposes only.*

Can be used to supply the full dependency information of a project in a single structured file.

Usage example: `ui5 serve -b static:/path/to/projectDependencies.yaml`  
`projectDependencies.yaml` contains something like:
````yaml
---
id: testsuite
version: "",
path: "./"
dependencies:
- id: sap.f
  version: "",
  path: "../sap.f"

- id: sap.m
  version: "",
  path: "../sap.m"
````

## Project Preprocessor
Enhances a given dependency tree based on a projects [configuration](#configuration).

### Enhanced dependency tree structure returned by the Project Preprocessor
````json
{
    "id": "projectA",
    "version": "1.0.0",
    "path": "/absolute/path/to/projectA",
    "specVersion": "0.1",
    "type": "application",
    "metadata": {
        "name": "sap.projectA",
        "copyright": "Some copyright ${currentYear}"
    },
    "resources": {
        "configuration": {
            "paths": {
                "webapp": "app"
            }
        },
        "pathMappings": {
             "/": "app"
        }
    },
    "dependencies": [
        {
            "id": "projectB",
            "version": "1.0.0",
            "path": "/path/to/projectB",
            "specVersion": "0.1",
            "type": "library",
            "metadata": {
                "name": "sap.ui.projectB"
            },
            "resources": {
                "configuration": {
                    "paths": {
                        "src": "src",
                        "test": "test"
                    }
                },
                "pathMappings": {
                    "/resources/": "src",
                    "/test-resources/": "test"
                }
            },
            "dependencies": [
                {
                    "id": "projectD",
                    "version": "1.0.0",
                    "path": "/path/to/different/projectD",
                    "specVersion": "0.1",
                    "type": "library",
                    "metadata": {
                        "name": "sap.ui.projectD"
                    },
                    "resources": {
                        "configuration": {
                            "paths": {
                                "src": "src/main/uilib",
                                "test": "src/test"
                            }
                        },
                        "pathMappings": {
                            "/resources/": "src/main/uilib",
                            "/test-resources/": "src/test"
                        }
                    },
                    "dependencies": []
                }
            ]
        },
        {
            "id": "projectD",
            "version": "1.0.0",
            "path": "/path/to/projectD",
            "specVersion": "0.1",
            "type": "library",
            "metadata": {
                "name": "sap.ui.projectD"
            },
            "resources": {
                "configuration": {
                    "paths": {
                        "src": "src/main/uilib",
                        "test": "src/test"
                    }
                },
                "pathMappings": {
                    "/resources/": "src/main/uilib",
                    "/test-resources/": "src/test"
                }
            },
            "dependencies": []
        }
    ]
}
````

## Configuration
### Project configuration
Typically located in a per-project `ui5.yaml` file.

#### Structure
##### YAML
````yaml
---
specVersion: "0.1"
type: application|library|custom
metadata:
  name: testsuite
  copyright: |-
   UI development toolkit for HTML5 (OpenUI5)
    * (c) Copyright 2009-${currentYear} SAP SE or an SAP affiliate company.
    * Licensed under the Apache License, Version 2.0 - see LICENSE.txt.
resources:
  configuration:
    paths:
        "<virtual path>": "<physical path>"
        "<virtual path 2>": "<physical path 2>"
  shims:
    some-dependency-name:
      type: library
      metadata:
        name: library-xy
      resources:
        configuration:
          paths:
            "src": "source"
server:
  settings:
    port: 8099
````

#### Properties
##### \<root\>
- `specVersion`: Version of the specification
- `type`: Either `application`, `library` or `custom` (custom not yet implemented). Defines default path mappings and build steps. Custom doesn't define any specific defaults

##### metadata
Some general information
- `name`: Name of the application/library/resource
- `copyright` (optional): String to be used for replacement of copyright placeholders in the project

##### resources (optional)
- `configuration`
    - `paths`: Mapping between virtual and physical paths
        + For type `application` there can be a setting for mapping the virtual path `webapp` to a physical path within the project
        + For type `library` there can be a setting for mapping the virtual paths `src` and `test` to physical paths within the project
- `shims`: Can be used to define, extend or override UI5 configs of dependencies. Inner structure equals the general structure. It is a key-value map where the key equals the project ID as supplied by the translator.

##### server (optional)
- `settings` (not yet implemented)
    - `port`: Project default server port. Can be overwritten via CLI parameters.

## Contributing
Please check our [Contribution Guidelines](https://github.com/SAP/ui5-tooling/blob/master/CONTRIBUTING.md).

## Support
Please follow our [Contribution Guidelines](https://github.com/SAP/ui5-tooling/blob/master/CONTRIBUTING.md#report-an-issue) on how to report an issue.

## License
This project is licensed under the Apache Software License, Version 2.0 except as noted otherwise in the [LICENSE](/LICENSE.txt) file.
