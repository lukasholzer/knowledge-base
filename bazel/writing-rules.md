# Writing Bazel rules
 
## Anatomy of the bazel rule

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

## Macros in combination with Rules

Often it is handy to wrap a rule in a macro to abstract some defaults away from your consumer.
This may help as well with copying files as it is not possible to copy files inside a rule.

You can only create new files in a rules but with a macro you can leverage the `copy_to_bin` rule
and chain multiple rules like here.


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

## How to traverse with aspects