# Nginx Gotchas & Non-Obvious Behavior

## Table of Contents
1. [How Nginx processes a request](#processing-a-request)
2. [The `if` Directive](#the-if-directive)
3. [proxy_pass trailing slash behavior](#proxy_pass-trailing-slash-behavior)
4. [Directive Inheritance](#directive-inheritance)
5. [Upstream Keepalive](#upstream-keepalive)

---
Beginner's guide is available at
[Beginner's guide](https://nginx.org/en/docs/beginners_guide.html).

## Processing a request

### Name-based virtual server

Nginx first decides which server should process the request. Let’s start with a simple configuration where all three virtual servers listen on port *:80:

```nginx
server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}
```

In this configuration nginx tests only the request’s header field “Host” to determine which server the request should be routed to. If its value does not match any server name, or the request does not contain this header field at all, then nginx will route the request to the default server for this port. In the configuration above, the default server is the first one — which is nginx’s standard default behaviour. It can also be set explicitly which server should be default, with the default_server parameter in the listen directive:

```nginx
server {
    listen      80 default_server;
    server_name example.net www.example.net;
    ...
}
```

The default_server parameter has been available since version 0.8.21. In earlier versions the default parameter should be used instead.
Note that the default server is a property of the listen port and not of the server name. 

You can find more information about this in [How nginx processes a request](https://nginx.org/en/docs/http/request_processing.html).

### Location evaluation order

As described in the [docs](https://nginx.org/en/docs/http/ngx_http_core_module.html#location), Nginx evaluates locations in special order, not necessarily the order locations are defined in config files. Nginx will first search for the most specific prefix location given by literal strings. The location with the longest prefix is selected and remembered. Then regular expressions are checked, in order they are defined in the config files. The regular expressions search terminates on the first match, and the corresponding location is used. If no regular expression match is found, the previously remembered longest prefix is used. Here are all the location modifiers:

- `=` -  exact match
- `~` - case sensitive regex match
- `~*` - case insensitive regex match
- `^~` - prefix match which prevents Regex expressions from being checked

Order of location match:

1. **`= /exact`** — Exact match. Wins immediately, stops all searching.
2. **`^~ /prefix`** — Preferential prefix. If matched, skips regex evaluation entirely.
3. **`~ regex`** / **`~* regex`** — Regular expressions, evaluated in config file order. First match wins.
4. **`/prefix`** — Standard prefix. Longest match wins, but only used if no regex matched.

Important caveats:
- A long prefix match does not beat a regex match (unless `^~` is used). 
- Nested `location` blocks are only evaluated if the outer block is the final match.
- `location /` is a catch-all prefix that matches everything, which means it's the lowest priority unless nothing else matches.


## The `if` Directive

**`if` inside `location` context is dangerous.** It creates an implicit nested location internally, which causes unexpected behavior with most directives.

### Safe inside `if` in location context:
- `return`
- `rewrite ... last` / `rewrite ... break`
- `set`

### Unsafe / unreliable inside `if`:
- `proxy_pass` (use `map` + named upstream or `set $backend` instead)
- `fastcgi_pass`
- `try_files`
- `alias`
- `root`
- Most other content-handler directives

### Pattern: Replace `if` with `map`

```nginx
# BAD
location / {
    if ($arg_backend = "v2") {
        proxy_pass http://backend_v2;
    }
    proxy_pass http://backend_v1;
}

# GOOD
map $arg_backend $target_backend {
    v2       http://backend_v2;
    default  http://backend_v1;
}

server {
    location / {
        proxy_pass $target_backend;
    }
}
```

Note: The address can be specified as a server group defined with a `upstream` directive. If proxy_pass is defined with a variable, like in above example, Nginx will check if it's specified as domain name, and search for it in the defined upstream server groups. If no server groups are find, it will resolve using a [resolver](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver).

## proxy_pass trailing slash behavior

The trailing slash on `proxy_pass` changes path handling:

```nginx
# no trailing slash - passes the full original URI
location /api/ {
    proxy_pass http://backend;
    # Request: /api/users/123 -> backend receives: /api/users/123
}

# trailing slash - strips and replaces the matched prefix
location /api/ {
    proxy_pass http://backend/;
    # Request: /api/users/123 -> backend receives: /users/123
}

# with trailing slash and prefix
location /api/ {
    proxy_pass http://backend/v2/;
    # Request: /api/users/123 -> backend receives: /v2/users/123
}
```


## Directive Inheritance

In some Nginx directives, such as add_header, proxy_set_header, and others, if both parent and child define the directive, the directives from the parent are ignored. 
Most commonly misunderstood with `proxy_set_header`:

```nginx
server {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    location /api/ {
        # Adding ONE proxy_set_header here WIPES ALL FOUR from server block
        proxy_set_header X-Custom "value";

        # Must re-declare all headers:
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://backend;
    }
}
```

Other commonly affected directives:
- `add_header`
- `grpc_set_header`
- `limit_conn`
- `limit_req`
- `add_trailer`
- `error_page`
- `proxy_hide_header`
- `fastcgi_param`
- `uwsgi_param`
- `scgi_param`
- `xslt_param`
- `xslt_string_param`
- `ssl_conf_command`
- `grpc_ssl_conf_command`
- `proxy_ssl_conf_command`
- `uwsgi_ssl_conf_command`
- `zone_sync_ssl_conf_command`
- `sub_filter`

Some directives can be specified multiple times in the same block and effectively add together. For example, you can have `proxy_next_upstream error;` in server block and `proxy_next_upstream timeout;` in the same block again. It would work the same as having `proxy_next_upstream error timeout;`. While it might not make sense having the same directive applied multiple times in the same block, it is useful in complex codebases where you have multiple includes, or a global include. For example there might be a file called `global-options` that is included in every server block and defines a `proxy_next_upstream error;` amongst opther options. If you want to use a another proxy_next_upstream value in one specific server block, you don't need to omit the global-options include and can just specify another proxy_next_upstream config. This is true for all directives that use `ngx_conf_set_bitmask_slot` in Nginx source.

```
http {
    proxy_next_upstream error;

    server {
        listen 80;
        server_name example.com;

        # This will cause nginx -t to complain about duplicate "error"
        # because "error" is already inherited from http level
        proxy_next_upstream error timeout;

        location /api {
            proxy_pass http://backend;
        }
    }
}
```




## Upstream Keepalive

The `keepalive` directive in `upstream` blocks is not the max number of connection, it is the **max number of idle connections** cached per worker process.

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;

    keepalive 32;              # Cache up to 32 idle connections per worker
    keepalive_requests 1000;   # Close connection after 1000 requests
    keepalive_time 1h;         # Max lifetime of a keepalive connection
    keepalive_timeout 60s;     # Idle timeout before closing cached connection
}
```

**Required additional directives** - without these, keepalive does nothing:
```nginx
location / {
    proxy_http_version 1.1;           # REQUIRED - default is 1.0 which uses Connection: close
    proxy_set_header Connection "";    # REQUIRED - clears the default "Connection: close" header
    proxy_pass http://backend;
}
```

Forgetting `proxy_http_version 1.1` or `proxy_http_version 2` and `Connection ""` is the #1 reason keepalive doesn't work. Always include both.