# Bazel query

The bazel query is used to query the dependency tree.

## Query for a macro

```python
bazel query "attr(generator_function, <macro-name>, //...)"
```