# Aspects

Aspects are a useful instrument when you need to collect data from your transitive dependencies of your target. 
For example you need a list of all package names that you depend on.

Let's assume we create a rule for yet another bundler and we need to train how to resolve custom module_names that are 
provided by `js_library` or `ts_library` dependencies. So we have synthetic module imports inside our code that references
tsconfig path aliases or custom module names like `import { foo } from '@myorg/package/bar';`. As this is no package inside our
`node_modules` our custom rule does not know on which barrel file or path it should resolve this module name.

To acomplish that in a normal rule we would loop over our dependencies and check if the provider is present like:

```python
for dep in ctx.attr.deps:
  if LinkablePackageInfo in dep:
    print(dep[LinkablePackageInfo].package_name)
```

But what if the package you need is not a direct dependency? Aspects for the rescue! With aspects it is possible to collect the information
off your dependencies like piggybacking on a dependency walker and collecting the needed information.

So first of all we need to define our aspect very similar to a rule:

```python
load("@build_bazel_rules_nodejs//:providers.bzl", "LinkablePackageInfo")

# Create a custom provider to access the collected information inside the rule
DepInfo = provider(
  fields = {
    'names': 'list of package names'
  }
)

def _dep_info_aspect_impl(target, ctx):
    """An aspect that collects a list of all transitive package names
    Args:
      target: The target, the aspect is being applied to.
      ctx: context that can be used to access attributes and generate outputs and actions.

    Returns:
      DepInfo provider with the collected package names
    """
    dep_list = []
    # Make sure the rule has a deps field to check its dependencies
    if hasattr(ctx.rule.attr, 'deps'):
        # Iterate through the deps like in a normal rule
        for dep in ctx.rule.attr.deps:
            # Get the linkable package info that is provided through a ts_library by setting the `module_name` field.
            if LinkablePackageInfo in dep:
                dep_list.append(dep[LinkablePackageInfo].package_name)

    return DepInfo(names = dep_list)

dep_info_aspect = aspect(
    implementation = _dep_info_aspect_impl,
    attr_aspects = ['deps'], # Here we are specifying a list of possible dependency fields where to look inside
)

```

After defining this aspect we now have to tell our rule how to collect the information:

```python

def _my_rule_impl(ctx):
    # now loop over our dependencies where the aspect is called on.
    for dep in ctx.attr.deps:
      # Now we can check if our dependency has transitive packages.
      if DepInfo in dep:
        print(dep[DepInfo].names)

  return []

my_rule = rule(
  attrs = {
     "deps": attr.label_list(
          allow_files = True,
          default = [],
          aspects = [dep_info_aspect], # pass here the aspect to tell bazel to look on all transitive packages
          doc = "List of dependencies",
      ),
  },
  implementation = _my_rule_impl,
)