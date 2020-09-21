# Patching

- [Patching](#patching)
  - [Patching bazel repositories](#patching-bazel-repositories)
  - [Patching bazel files that are managed through npm](#patching-bazel-files-that-are-managed-through-npm)
    - [Sample patch for an npm package](#sample-patch-for-an-npm-package)

> This section is for all of you that are living on the bleeding edge 💉

Often it is necessary to patch a small use case for a repository as they have not released, or a fix you have already created a Pull-Request for with the desired fix.

For this bazel allows you to patch external repositories.

## Patching bazel repositories

When patching bazel repositories that are installed via a `http_archive` you can simply add the `patches` property with the paths.

```python
http_archive(
    name = "build_bazel_rules_nodejs",
    sha256 = RULES_NODEJS_SHA256,
    url = "https://github.com/bazelbuild/rules_nodejs/releases/download/%s/rules_nodejs-%s.tar.gz" % (RULES_NODEJS_VERSION, RULES_NODEJS_VERSION),
    patches = [
      "//:rules_nodejs-js-named-module-info.patch",
      "//:rules_nodejs-npm-ci.patch"
    ]
)
```

You can create a patch by performing following actions:

1. going to the repository (clone the repository first)
2. checkout the tag you are using
3. perform the change you want
4. run `git diff > ~/path/to/your-patch-file.patch`
5. remove the `a/` and `b/` occurrences in the paths.
6. place the generated file into the repository and check it in

**NOTE**: If you add files they won't occur in the diff as they are untracked. In this case just add all files with `git add .` and then run `git diff --cached  > ~/path/to/your-patch-file.patch`. 

A patch file can look like this:

```patch
diff --git internal/npm_install/npm_install.bzl internal/npm_install/npm_install.bzl
index 7bbe1297..43ecf2ec 100644
--- internal/npm_install/npm_install.bzl
+++ internal/npm_install/npm_install.bzl
@@ -195,7 +195,7 @@ def _npm_install_impl(repository_ctx):
     is_windows_host = is_windows_os(repository_ctx)
     node = repository_ctx.path(get_node_label(repository_ctx))
     npm = get_npm_label(repository_ctx)
-    npm_args = ["install"] + repository_ctx.attr.args
+    npm_args = ["ci"] + repository_ctx.attr.args

     # If symlink_node_modules is true then run the package manager
     # in the package.json folder; otherwise, run it in the root of
```

When you run the next bazel command it will automatically apply your patch.

## Patching bazel files that are managed through npm

If you want to apply a patch in a repsoitory that is managed by npm like `@bazel/typescript` you can leverage the `patch-package` npm package.

1. Install the package with `npm i -D patch-package`. If you have to patch something in your shipped dependencies then remove the `-D` in the install command.
2. Go to the `node_modules` folder inside you package and apply your changes.
3. Run `npx patch-package ${package-name}` this will create a folder `patches` with your patched changes.
4. Add the `patch-package` to the postinstall script of your package. If you need to patch a dependency that is used in production as well be sure to include the postinstall in the shipped `package.json` as well.

### Sample patch for an npm package

```patch
diff --git a/node_modules/@bazel/typescript/internal/common/tsconfig.bzl b/node_modules/@bazel/typescript/internal/common/tsconfig.bzl
index fc1d6f7..9af2e92 100755
--- a/node_modules/@bazel/typescript/internal/common/tsconfig.bzl
+++ b/node_modules/@bazel/typescript/internal/common/tsconfig.bzl
@@ -264,7 +264,7 @@ def create_tsconfig(
         # We don't support this compiler option (See github #32), so
         # always emit declaration files in the same location as outDir.
         "declarationDir": "/".join([workspace_path, outdir_path]),
-        "stripInternal": True,
+        "stripInternal": False,
 
         # Embed source maps and sources in .js outputs
         "inlineSourceMap": True,

```