
# Anatomy of the bazel rule

The rule consists out of an implementation and attributes.

```python

my_rule = rule(
    implementation = _my_rule_impl,
    attrs = {
        "data": attr.label_list(
            allow_files = True,
            default = [],
            doc = "Static data",
        ),
    },
```


# Copy Files with macro


```python
def my_rule_macro(name = "default", deps = [], **kwargs):

    # Copy assets
    copy_to_bin(
        name = "copy_%s_assets" % name,
        srcs = assets,
    )

    extra_deps = ["@npm//@rollup/plugin-node-resolve"]

    rollup(
        name = name,
        deps = deps + extra_deps,
        data = [":copy_%s_assets" % name],
        **kwargs
    )

```

# How to traverse with aspects