# Learning HAProxy with Docker — A Hands-On Guide

## Why HAProxy?

HAProxy (High Availability Proxy) is a TCP/HTTP load balancer and reverse proxy. It sits in front of your servers and decides which backend gets each request, based on rules you define. It's used heavily in production because it's fast, single-purpose, and extremely configurable — not a general webserver like Nginx or Apache, but a dedicated traffic router.

Core jobs it does:̦
- **Load balancing** — spread traffic across multiple servers
- **Health checking** — stop sending traffic to servers that are down
- **SSL termination** — decrypt HTTPS at the edge so backends don't have to
- **Reverse proxying** — hide your real servers behind one address
- **Traffic shaping** — rate limiting, sti̦̦cky sessions, routing by path/header

## Why HAProxy Exists

HAProxy was created **only** to solve one problem:

> Route traffic as fast, reliably and intelligently as possible.

Everything in HAProxy is optimized around that goal.

NGINX was created as a **web server**, and reverse proxy/load balancing were added later.

That's why NGINX has features HAProxy never will, such as:

- Serving HTML/CSS/JS
- Caching
- Gzip/Brotli compression
- Directory listings
- URL rewrites
- PHP/FastCGI support
- Static content delivery

HAProxy intentionally avoids these responsibilities.

Test load balancing:

```bash
for i in {1..6}; do curl -s http://localhost:8080; done
```

You should see the response cycle through WEB1 → WEB2 → WEB3 → WEB1... That's `roundrobin` in action.

Open the stats dashboard in a browser: `http://localhost:8404/stats` — you'll see live status, request counts, and health check state for each server.

---

## Key Concepts to Understand

### 1. Frontend vs Backend
- **Frontend**: where HAProxy listens for incoming connections (`bind *:80`). Defines *what* traffic to accept and *where to send it*.
- **Backend**: the pool of real servers HAProxy forwards traffic to, plus the algorithm for picking one.

Think of frontend as "the door" and backend as "the rooms behind it."

### 2. Modes: `http` vs `tcp`
- `mode http` — HAProxy understands HTTP headers, paths, cookies. Needed for Layer 7 routing (routing by URL path, header, hostname).
- `mode tcp` — Layer 4, just forwards raw connections (used for databases, non-HTTP protocols, or when you want minimal overhead).

### 3. Load balancing algorithms (`balance`)
- `roundrobin` — cycles evenly (what we used above)
- `leastconn` — sends to the server with fewest active connections (great for long-lived connections)
- `source` — hashes client IP so the same client always hits the same server (basic session stickiness without cookies)
- `uri` — hashes the URL, useful for cache servers

### 4. Health checks
```
option httpchk GET /health
server web1 web1:80 check inter 2000 rise 2 fall 3
```
- `check` enables active health checking
- `inter 2000` — check every 2000ms
- `fall 3` — mark down after 3 consecutive failures
- `rise 2` — mark back up after 2 consecutive successes

This is one of HAProxy's most valuable features: it automatically stops routing to a broken server without you doing anything.

### 5. ACLs (Access Control Lists) — routing logic
This is where HAProxy becomes powerful. ACLs let you route based on conditions:

```
frontend http_front
    bind *:80
    acl is_api path_beg /api
    acl is_static path_beg /static
    use_backend api_backend if is_api
    use_backend static_backend if is_static
    default_backend web_servers
```

You can match on path, headers, hostname (`Host` header), cookies, source IP, and more.

### 6. Sticky sessions (session persistence)
Two common approaches:
```
backend web_servers
    balance roundrobin
    cookie SERVERID insert indirect nocache
    server web1 web1:80 check cookie web1
    server web2 web2:80 check cookie web2
```
HAProxy injects a cookie so a client always returns to the same backend — useful when backend servers hold local session state.

### 7. SSL termination
```
frontend https_front
    bind *:443 ssl crt /usr/local/etc/haproxy/certs/combined.pem
    default_backend web_servers
```
HAProxy decrypts HTTPS here; traffic to backends can then be plain HTTP, simplifying certificate management to one place.

---

## Tips and Tricks

1. **Always test config before reloading in production:**
   ```bash
   docker exec <container> haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
   ```
   `-c` validates syntax without starting the process — saves you from a bad reload taking down traffic.

2. **Use the stats page aggressively while learning.** `stats admin if TRUE` (as above) lets you manually mark a server up/down from the dashboard — great for testing failover behavior without stopping containers.

3. **Simulate a server failure** to watch health checks work:
   ```bash
   docker compose stop web2
   ```
   Watch the stats dashboard — web2 turns red/down within a few health-check intervals, and traffic quietly reroutes to web1/web3 only.

4. **Watch logs live:**
   ```bash
   docker compose logs -f haproxy
   ```
   With `option httplog`, every request is logged with status code, timing, and which backend server handled it — extremely useful for debugging routing decisions.

5. **Rate limiting / abuse protection** using stick-tables:
   ```
   frontend http_front
       stick-table type ip size 100k expire 30s store http_req_rate(10s)
       http-request track-sc0 src
       http-request deny if { sc_http_req_rate(0) gt 20 }
   ```
   This blocks any client IP making more than 20 requests in 10 seconds.

6. **Graceful reloads matter.** HAProxy supports zero-downtime reloads (`-sf` seamless flag under the hood) — this is one reason it's chosen over restarting Nginx for high-traffic routing changes. In Docker, `docker kill -s HUP <container>` triggers this if the image supports it.

7. **Keep timeouts explicit.** Missing `timeout connect/client/server` values are a classic source of hung connections. HAProxy will actually refuse to start (or warn loudly) without them in `defaults`.

8. **Separate the stats port from your main traffic port** (like we did with 8404 vs 80) and don't expose it publicly in real deployments — it reveals backend server names/IPs.

9. **Use `option forwardfor`** so backend servers can see the real client IP instead of HAProxy's:
   ```
   option forwardfor
   ```
   Then read `X-Forwarded-For` in your backend app logs.
10. **Alway have one line brake or empty line at the end of haproxy.cfg file**. If missing This might throw parsing error: Missing LF on last line, file might have been truncated at position x

---

## FAQ

```markdown
## Frequently Asked Questions

### What is the difference between HAProxy and NGINX?

HAProxy is a dedicated load balancer and reverse proxy.

NGINX is primarily a web server that also provides reverse proxy and load balancing capabilities.

---

### When should I use `mode tcp`?

Use TCP mode for protocols that are not HTTP, such as:

- MySQL
- PostgreSQL
- Redis
- SSH
- MQTT

HAProxy forwards raw TCP connections without inspecting application data.

---

### When should I use `mode http`?

Use HTTP mode when you need to route based on:

- URL path
- Host header
- Cookies
- HTTP headers
- Request method

---

### What is the difference between a frontend and a backend?

Frontend:
- Accepts incoming client connections.
- Decides where requests should go.

Backend:
- Contains one or more application servers.
- Defines how requests are distributed.

---

### What does the `check` keyword do?

It enables active health checking.

If a server becomes unhealthy, HAProxy automatically removes it from the load-balancing pool.

---

### Why does HAProxy return a 204 for OPTIONS requests?

These are CORS preflight requests.

HAProxy can respond directly without forwarding the request to the application, reducing unnecessary load.

---

### Why do I see only one backend server receiving requests?

Possible reasons include:

- Sticky sessions
- `source` balancing
- Browser caching
- Only one healthy backend
- Testing with a single persistent connection

---

### Can HAProxy serve HTML, CSS, or images?

No.

HAProxy is not a web server.

Use NGINX or Apache for serving static content.

---

### How do I reload HAProxy without downtime?

Inside Docker:

```bash
docker kill -s HUP haproxy