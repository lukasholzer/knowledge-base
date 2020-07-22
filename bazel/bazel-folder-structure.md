# Bazel locations, workspaces and repositories

## Bazel home

Holds the content of a workspace (marked by the existance of a file called _WORKSPACE_)

Bazel symlinks your code project into:

`Users/user/.cache/bazel/hash/**execroot/<workspace-name>**`

<workspace-name> = determined in the WORKSPACE file

*Note: This path is different for every OS, the provided example is from an Ubuntu 18.04 machine*


## Bazel out

Holds build target outputs, also called bazel-bin, depending on OS 

`/Users/user/.cache/bazel/hash/**execroot/<workspace-name>/bazel-out/k8-fastbuild**`


Per default, Bazel creates a symlink to Bazel out in the root of any workspace

Get details via:

```bash
bazel info
```

## Bazel external folders

`/Users/user/.cache/bazel/hash/**external**`

Holds many many different repositories for third party dependencies
All those repositories are specified in the WORKSPACE file in the root of your project

Example:
```python
# in WORKSPACE file 

http_archive(
    name = "build_bazel_rules_nodejs",
    sha256 = ..., # some hash
    url = "http://..." # some github 
)
```
creates (downloads repository into):
```bash
 -external
  |
  | - build_bazel_rules_nodejs
  | _ _ rule.bzl #(exports 'myrule')
```
Can be referenced via 
```python
# this is some random BUILD.bzl file

# @build_bazel_rules_nodejs -> is a repository
load(@build_bazel_rules_nodejs//rule.bzl, "myrule")
```

## npm_modules:

npm_modules are downloaded in WORKSPACE file by repo ruled for 'npm install'

this creates a _npm_ repository

this means we can reference third party dependencies via:
```python
# some build rule
build_angular(
    name="foo",
    deps=[
        # @npm// -> repository name defined by 'npm install' repository rule
        # @angular/core -> name of package as specified in package.json
        "@npm//@angular/core",
    ]

)
```
