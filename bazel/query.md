# Bazel query

The bazel query is used to query the dependency tree.

## Query for a macro

```python
bazel query "attr(generator_function, <macro-name>, //...)"
```

## Run a Query Output

If you want to run the result of a query with a build or test you can leverage the `xargs`.
This will pass the output to the `bazel test` or `build`.

```python
bazel query "attr(generator_function, <macro-name>, //...)" | xargs bazel test
```