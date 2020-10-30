# Writing Bazel rules

First of all I might need to mention that writing a custom rule should be the last solution on trying to get a package managed by bazel. Always try to use existing rules for your problem. If there is no exisitng rule try to leverage a CLI tool that can be wrapped by a macro.

Writing a rule is needed if you have to collect different parts of your dependencies and massage them into a format that your tool can understand it.

For testing reasons I always can recommend putting as less logic as possible inside Starlark rules and try to create small nodejs helper tools that are doing the job for you.

- [Writing Bazel rules](#writing-bazel-rules)
  - [Concepts in Bazel](#concepts-in-bazel)
    - [Structs](#structs)
    - [Providers](#providers)
  - [Anatomy of the bazel rule](#anatomy-of-the-bazel-rule)
  - [Macros in combination with Rules](#macros-in-combination-with-rules)

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

### Providers

Providers can be understood like a label to identify a specific content that was stored by a dependency. 
Think about a post office that needs to identify the packages to deliver them correctly. The provider would be the label on the package.

You can use already defined providers to get specific information like accessing the type declaration information of your dependency.

```python
load("@build_bazel_rules_nodejs//:providers.bzl", "DeclarationInfo")

# Loop inside your rule over the dependencies and collect the type declarations.
for dep in ctx.attr.deps:
    if DeclarationInfo in dep:
        print(dep[DeclarationInfo].declarations.to_list())
```

But you can create your own providers as well, the most common use case would be if you are writing an aspect to collect specific data.
You can read more about on the [aspects page](./aspects.md). 

```python
# Mark that the naming convention of a provider is to end with `Info`
CustomProviderInfo = provider(
  fields = {
    # specify a list of fields that a provider can have like the dependencies field on the DeclarationInfo provider
    'my_field': 'Holds custom information'
  }
)
```

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
