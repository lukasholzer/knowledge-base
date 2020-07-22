# Building an angular application

- [Building an angular application](#building-an-angular-application)
  - [TLDR;](#tldr)
  - [Building angular modules](#building-angular-modules)
    - [Prerequisites the custom compiler binary ngtsc](#prerequisites-the-custom-compiler-binary-ngtsc)
    - [Building the first module](#building-the-first-module)
      - [Dependencies](#dependencies)
      - [tsconfig](#tsconfig)
      - [Styles](#styles)
      - [Assets](#assets)
  - [Serve an angular application](#serve-an-angular-application)
    - [prerequisite](#prerequisite)
  - [Bundling an angular application](#bundling-an-angular-application)
    - [Minification](#minification)
    - [Generate bundle for differential loading](#generate-bundle-for-differential-loading)

## TLDR;

Use the `ts_library(use_angular_plugin=True)` that is using the `ngtsc`. With that you can compile modules and your angular application.
After that you have to create bundles with **rollup** for the main file in an `es2015` format and with babel an `es5` version for differential loading.
All the minification and optimization (Angular build optimizer) has to be done on their own.
Last but not least the application can be packaged using the `pkg_web` rule.

[API documentation of the ts_library][1]

## Building angular modules

To build angular modules for Ivy (the recommended way). You should leverage the `ts_library` rule that is loaded from the `@bazel/typescript` npm package.
You have to create a custom compiler binary that can be used instead of the plain tsc, that is the default of the `ts_library` rule.

### Prerequisites the custom compiler binary ngtsc

The custom ts_library compiler that is defined below runs the tsc_wrapped with the statical linked angular compiler cli (`ngtsc`).
Just place this in a build file inside the repository.

```python
# BUILD.bazel
load("@build_bazel_rules_nodejs//:index.bzl", "nodejs_binary")

package(default_visibility = ["//:__subpackages__"])

nodejs_binary(
    name = "tsc_wrapped_with_angular",
    data = [
        "@npm//@angular/compiler-cli",
        "@npm//@bazel/typescript",
    ],
    entry_point = "@npm//:node_modules/@bazel/typescript/internal/tsc_wrapped/tsc_wrapped.js",
    visibility = ["//visibility:public"],
)
```

After that you can create a macro that is using the `ts_library` with the custom compiler binary that has the compiler cli statically linked.

```python
# ng_ts_library.bzl
load("@npm//@bazel/typescript:index.bzl", "ts_library")

def ng_ts_library(deps = [], **kwargs):
    ts_library(
        compiler = "//path/to:tsc_wrapped_with_angular",
        supports_workers = True,
        use_angular_plugin = True,
        deps = deps + [
            "@npm//@angular/compiler-cli",
            "@npm//tslib"
        ],
        **kwargs
    )

```

### Building the first module

When you start building an angular application with bazel you cannot start at the `app.module.ts` or the `main.ts` file. You have to bazelify first all of its dependencies. This means you have to start at the very bottom of your dependency tree. Look for the loose modules (mostly the lazy loaded ones). And start building them.

```python
load("//path/to/macro:ng_ts_library.bzl", "ng_ts_library")
load("@io_bazel_rules_sass//:defs.bzl", "sass_binary")

ng_ts_library(
    name = "feature-module",
    srcs = glob(
        include = [ "*.ts" ],
        exclude = [ "*.spec.ts" ],
    ),
    angular_assets = [
        ":styles",
        "feature.component.html"
    ],
    tsconfig = "//apps/my-application:tsconfig-app",
    deps = [
        "@npm//@angular/common",
        "@npm//@angular/core",
        "@npm//@angular/router",
        "@npm//rxjs",
    ],
)

sass_binary(
    name = "styles",
    src = "feature.component.scss",
)
```

Your first `BUILD.bazel` file inside the lazy loaded feature module could look like this. There are several things that I want to describe below:

1. Dependencies
2. tsconfig
3. Styles
4. Assets

#### Dependencies

With bazel you have to care and manage your dependencies. This means that all of your imports in the typescript files are dependencies of the compilation.
If it is a dependency inside your application that is not part of the specified glob inside the `srcs` then you have to specify the matching build target for the part of the application. If it is an **npm dependency** specify the package name after the `@npm//...`. You don't have to specify transitive dependencies.

#### tsconfig

Every Angular application has its own tsconfig. If you don't extend from the tsconfig with the `"extends": "../../tsconfig.json"` pattern then you can specify there a direct file path (In this case be sure that the file will be exported in the matching BUILD file `exports_files(["tsconfig.app.json"])`).

If you extend in your config you have to declare the tsconfig as build target with it's dependencies.

```python
load("@npm//@bazel/typescript:index.bzl", "ts_config")

ts_config(
    name = "tsconfig-app",
    src = ":tsconfig.app.json",
    deps = [
        ":tsconfig.json",
        "//:tsconfig.json",
    ],
)
```

#### Styles

If your part of the application that you want to build with the ngtsc is depending on some styles that are not `.css` file then you have to preprocess them. In our case we are using `scss` so we need to compile the scss before we can pass it as an asset to the angular compiler.

TODO: write about scss compilatioin


#### Assets

You have to provide all the assets (`.html` and `.css`) for the template compilation of the angular compiler. Note that you have to preprocess other style formats than `.css`

## Serve an angular application

To serve an application with a dev server similar to the `ng serve my-application` bazel provides the `ts_devserver` rule from the `@bazel/typescript` npm package.

### prerequisite

The ts_devserver is using [requirejs][4] under the hood to load all the previously compiled modules and serve them for the dev server. As requirejs is using the AMD module resolution. This module resolution is similar to the UMD modules that are provided by the devmode compilation of the ts_library. 

A sample how the AMD module resolution looks like:

```javascript
//Calling define with a dependency array and a factory function
define(['modulenamesample', 'dynatrace/libs/otherlib'], function (dep1, dep2) {
    //Define the module value by returning a value.
    return function () {};
});
```

But to be compatible with the UMD/AMD format we need a shim for `rxjs` to be compatible with this file format [rxjs_shims.js](../assets/rxjs_shims.js).


```python
load("@npm//@bazel/typescript:index.bzl", "ts_devserver")

ts_devserver(
    name = "devserver",
    additional_root_paths = [
        "npm/node_modules",
        "npm/node_modules/@dynatrace/barista-fonts",
    ],
    entry_module = "dynatrace/apps/components-e2e/src/main.dev",
    port = 4200,
    scripts = [
        "@npm//:node_modules/tslib/tslib.js",
        "//tools/bazel_rules/dev_server:rxjs_umd_modules",
    ],
    serving_path = "/bundle.js",
    static_files = [
        ":index.html",
        ":styles.css",
        "@npm//:node_modules/zone.js/dist/zone.min.js",
    ],
    deps = [
        ":src",
        "@npm//@dynatrace/barista-fonts",
        "@npm//@dynatrace/barista-icons",
    ],
)

```

## Bundling an angular application

After the `main.ts` where the bootstrapping is done is built with the ngtsc you can start bundling your application.
For bundling an application use [rollup][2] from the `@bazel/rollup` package.

```python
load("@npm//@bazel/rollup:index.bzl", "rollup_bundle")

rollup_bundle(
    name = "bundle-es2015",
    config_file = "//apps/my-app:rollup.config.js",
    entry_points = {
        "main.prod.ts": "index",
    },
    output_dir = True,
    sourcemap = "false",
    deps = [":src"],
)
```

### Minification

To minify the bundle size and uglify the javascript you can leverage [terser][3] from the `@bazel/terser` package.

```python
load("@npm//@bazel/terser:index.bzl", "terser_minified")

terser_minified(
    name = "bundle-es2015.min",
    src = ":bundle-es2015",
    sourcemap = False,
)
```

### Generate bundle for differential loading

To generate **es5** bundles for differential loading that can be used for legacy browsers with the `nomodule` attribute. We are leveraging babel.
install the `@babel/cli` and use it directly with bazel to downlevel the previously generated bundle of the `es2015` aka `es6` version.

```python
load("@npm//@babel/cli:index.bzl", "babel")

babel(
    name = "bundle-es5",
    args = [
        "$(execpath :bundle-es2015)",
        "--no-babelrc",
        "--source-maps",
        "--presets=@babel/preset-env",
        "--out-dir",
        "$(@D)",
    ],
    data = [
        ":bundle-es2015",
        "@npm//@babel/preset-env",
    ],
    output_dir = True,
)
```


```python
# ============================================================
#
# CONVENTIONAL BUNDELING
#
# ============================================================

_ASSETS = [
    ":styles.css",
    "@npm//:node_modules/zone.js/dist/zone.min.js",
]

rollup_bundle(
    name = "bundle-es2015",
    config_file = "//apps/components-e2e:rollup.config.js",
    entry_points = {
        "main.prod.ts": "index",
    },
    output_dir = True,
    sourcemap = "false",
    deps = [":src"],
)

babel(
    name = "bundle-es5",
    args = [
        "$(execpath :bundle-es2015)",
        "--no-babelrc",
        "--source-maps",
        "--presets=@babel/preset-env",
        "--out-dir",
        "$(@D)",
    ],
    data = [
        ":bundle-es2015",
        "@npm//@babel/preset-env",
    ],
    output_dir = True,
)

terser_minified(
    name = "bundle-es2015.min",
    src = ":bundle-es2015",
    sourcemap = False,
)

terser_minified(
    name = "bundle-es5.min",
    src = ":bundle-es5",
    sourcemap = False,
)

html_insert_assets(
    name = "inject_scripts_for_prod",
    #     # we can't output "src/example/index.html" since that collides with the devmode file.
    #     # pkg_web rule will re-root paths that start with _{name} by default
    #     # so we output "_prodapp/src/example/index.html" so that it is mapped to
    #     # `example/index.html` in the web package.
    outs = ["index.html"],
    args = [
        "--html=$(execpath :index.prod.html)",
        "--out=$@",
        "--roots=. $(RULEDIR)",
        "--assets",
    ] + ["$(execpath %s)" % s for s in _ASSETS],
    data = _ASSETS + [
        ":index.prod.html",
    ],
)
```

[1]: https://bazelbuild.github.io/rules_nodejs/TypeScript.html#ts_library "ts_library API documentation"
[2]: https://rollupjs.org/guide/en/ "A lightweight javascript module bundler"
[3]: https://github.com/terser/terser "JavaScript minifier and mangler"
[4]: https://requirejs.org/ "Require JS a JavaScript module loader"