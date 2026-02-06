# OpenResty / Lua-nginx-module API Reference

## Table of Contents
1. [Phase handlers & execution order](#phase-handlers--execution-order)
2. [API Availability by Phase](#api-availability-by-phase)
4. [Shared Dictionary](#shared-dictionary)
5. [Request Body Handling](#request-body-handling)
6. [Subrequests](#subrequests)
7. [ngx.ctx vs ngx.var](#ngxctx-vs-ngxvar)

---

## Phase handlers & execution order

OpenResty/lua-nginx-module phases execute in this order:

```
init_by_lua*               - Main process startup (runs only once)
init_worker_by_lua*        - Runs on worker startup, once per worker
exit_worker_by_lua*        - Runs on worker exit, once per worker
ssl_certificate_by_lua*    - SSL handshake (before HTTP parsing)
set_by_lua*                - assign a value to a variable
rewrite_by_lua*            - URI rewrite phase
access_by_lua*             - Access control phase
content_by_lua*            - Content generation
balancer_by_lua*           - Custom upstream balancing (inside upstream block)
header_filter_by_lua*      - Response header modification
body_filter_by_lua*        - Response body modification (called per chunk)
log_by_lua*                - Logging phase, runs after response sent to client
```

**Notes:**
- `content_by_lua*` is mutually exclusive with `proxy_pass`, `fastcgi_pass`, etc. You cannot use both in the same location. To modify proxied responses, use `header_filter_by_lua*` and `body_filter_by_lua*`.
- `body_filter_by_lua*` is called **multiple times per request**, once per response chunk. Use `ngx.arg[2]` (eof flag) to detect the last chunk.
- `set_by_lua*` runs in the Nginx rewrite phase, but blocks the event loop, so the code should be short and fast
- `log_by_lua*` runs after the response is sent — cosockets are available but the response is already committed.

## API availability by phase

- init_by_lua:
    - Only Loggin APIs (ngx.log and print) and shared dictionary (ngx.shared.DICT) are supported in this phase
- exit_worker_by_lua:
    - Cannot use ngx.timer, because it runs after all timers have been processed
- set_by_lua:
    - Disabled Output API functions (e.g. ngx.say and ngx.send_headers)
    - Disabled Control API functions (e.g., ngx.exit)
    - Disabled Subrequest API functions (e.g., ngx.location.capture and ngx.location.capture_multi)
    - Disabled Cosocket API functions (e.g., ngx.socket.tcp and ngx.req.socket).
    - Disabled Sleeping API function ngx.sleep.
- header_filter_by_lua:
    - Disabled Output API functions (e.g. ngx.say and ngx.send_headers)
    - Disabled Control API functions (e.g., ngx.exit)
    - Disabled Subrequest API functions (e.g., ngx.location.capture and ngx.location.capture_multi)
    - Disabled Cosocket API functions (e.g., ngx.socket.tcp and ngx.req.socket).
- body_filter_by_lua:
    - Disabled Output API functions (e.g. ngx.say and ngx.send_headers)
    - Disabled Control API functions (e.g., ngx.exit)
    - Disabled Subrequest API functions (e.g., ngx.location.capture and ngx.location.capture_multi)
    - Disabled Cosocket API functions (e.g., ngx.socket.tcp and ngx.req.socket).
- log_by_lua:
    - Disabled Output API functions (e.g. ngx.say and ngx.send_headers)
    - Disabled Control API functions (e.g., ngx.exit)
    - Disabled Subrequest API functions (e.g., ngx.location.capture and ngx.location.capture_multi)
    - Disabled Cosocket API functions (e.g., ngx.socket.tcp and ngx.req.socket).


**The most common mistakes:** 
- Trying to use cosockets or subrequests in `header_filter_by_lua*` or `body_filter_by_lua*`. These phases do NOT support I/O. 
- Trying to change caching behavior in header_filter_by_lua. It runs after caching has been done, so the only thing you can do is bypass, but cannot change TTL, etc.


## Shared Dictionary

`lua_shared_dict` provides cross-worker shared memory with atomic operations.

```nginx
# declare shared dict and its size in http block
lua_shared_dict my_cache 10m;
```

```lua
local cache = ngx.shared.my_cache

-- Basic operations
cache:set("key", "value", exptime_seconds)  -- exptime 0 = no expiry
cache:get("key")
cache:delete("key")

-- Atomic counter operations
cache:incr("counter", 1, 0)          -- incr(key, increment value, init value). Init value is used if key doesn't exist
                                      
cache:incr("counter", 1, 0, exptime) -- 4th argument is init_ttl, which is expiration time for key, but is only set on init, not on every incr.

-- List all keys, don't use in production, because it locks the dictionary and blocks Nginx worker processes trying to access said dictionary
local keys = cache:get_keys(0)  -- 0 = all keys, or pass max count
```

**Capacity notes:**
- When the dict is full, `set()` evicts LRU entries
- `add()` is same as set, but only sets key if it does not exist.
- Values are limited to strings and numbers. For tables, use `cjson.encode()`/`cjson.decode()`.

## Request Body Handling

**You must explicitly read the body before accessing it:**

```lua
ngx.req.read_body()

local body = ngx.req.get_body_data()

-- If body was too large for memory buffer, it's in a temp file
if not body then
    local file = ngx.req.get_body_file()
    if file then
        local f = io.open(file, "r")
        body = f:read("*a")
        f:close()
    end
end
```

## Subrequests

`ngx.location.capture` and `mirror` makes internal Nginx subrequests (not external HTTP calls):

Common issue with subrequest is variable handling. Maps that are not volatile get evaluated once and the value gets cached, while when using set this is not true, as it gets evaluated in subrequest again. For example, if you use `set $my_var = 1` in the main request, modify it in Lua `ngx.var.my_var=2`, a subrequest will run the `set` again and then the value of `$my_var` will be 2 in main request but 1 in subrequest. Instead, if you define the variable via map, subrequest won't re-evaluate the map, and even if you modify the variable value in Lua in main request, the subrequest will have the same value of the variable as the main request has. 

## ngx.ctx vs ngx.var

| Feature              | `ngx.ctx`                  | `ngx.var`                     |
|----------------------|----------------------------|-------------------------------|
| Type                 | Lua table                  | String-only                   |
| Performance          | Fast (Lua table lookup)    | Slow (crosses Lua/C boundary) |
| Scope                | Per-request, all phases    | Per-request, all phases       |
| In subrequests       | Separate copy              | Shared with parent            |
| In timers            | Empty (no request context) | Unavailable                   |
| Needs declaration    | No                         | Yes (`set $var ""` in config) |
| Complex values       | Tables, functions, etc.    | Strings only                  |

**Best practice:** Use `ngx.ctx` for passing data between Lua phases. Use `ngx.var` only when you need to interact with Nginx config directives or need the variable in non-Lua parts of the config (like `log_format`).
