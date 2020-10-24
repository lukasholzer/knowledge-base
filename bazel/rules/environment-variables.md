# Environment Variables

Add your environment variables or secrets to the .bazelrc file

```bash
build --action_env=MY_ENV_VARIABLE=the-value-of-it
```

Then It can be passed to the nodejs binary via:

```python
my_variable = ctx.configuration.default_shell_env["MY_ENV_VARIABLE"]
ctx.actions.run(
  # ...
  env = {
    "MY_ENV_VARIABLE": my_variable,
    "OTHER_ENV_VAR": 'my-value',
  }
)
```