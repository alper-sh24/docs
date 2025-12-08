# Using self-hosted ollama as VSCode agent

If you have a self-hosted ollama instance, whether in a local or a public server, you can use that as your AI agent in VSCode.

## Continue extension

https://marketplace.visualstudio.com/items?itemName=Continue.continue

The **Continue** extension allows you to define custom providers and models to use as your AI helper. There may be other extensions that also have this feature, but this one was a popular recommendation on the internet.

Local config:

```yaml
name: Local Config
version: 1.0.0
schema: v1
models:
  - name: Ollama Remote
    provider: ollama
    model: AUTODETECT
    apiBase: "http://10.10.12.210:11434"
```

After you configure it, you can easily start using the local AI model provided by your self-hosted ollama instance.
