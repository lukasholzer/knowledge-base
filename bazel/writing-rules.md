# Writing Bazel rules

To write custom rules in bazel we need to get some concepts like structs, providers and depsets

## Concepts in Bazel

### Structs

A struct is a generic object with fields that is provided by Bazel it is not part of the Starlark language.
A struct value is a tuple with a name for each value. 

```python
s = struct(
        x = 2, 
        y = 3,
    )
```

The fields of a struct can be accessed like fields in an object in Python.

```python
return s.x + getattr(s, "y")  # returns 5
```

In addition you can get all keys with the `dir` function. If you need a JSON representation you 
can call the `to_json` method on it.

```python
struct(key=123).to_json()
# {"key":123}
```

https://docs.bazel.build/versions/master/skylark/lib/struct.html

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