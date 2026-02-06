# Testing & Validation Patterns

## Table of Contents
1. [Config Validation](#config-validation)

---

## Config Validation

### Basic validation
```bash
# Tests the config. It does not detect Lua issues, those show in runtime.
nginx -t
nginx -t -c /path/to/custom/nginx.conf

# Dump the full resolved config (does not resolve includes)
nginx -T

```

### Pre-deployment checklist
Before deploying a config change, verify:
1. `nginx -t` passes
2. Reload (not restart) to apply: `service nginx reload` preserves existing connections

### Graceful reload behavior
`nginx reload` does the following:
1. Master process reads new config
2. Starts new worker processes with new config
3. Sends graceful shutdown signal to old workers
4. Old workers finish existing requests (up to `worker_shutdown_timeout`)
5. Old workers exit

If the new config has a syntax error, the reload is rejected entirely — old workers continue unaffected. However, if the config is syntactically valid but semantically broken (e.g., upstream doesn't exist), the reload succeeds but new workers may fail to proxy.
