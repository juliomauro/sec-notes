# The Header That Opened the Network

> **Category:** Linux · Web · Lateral Movement  
> **Level:** Practitioner  

---

There is a class of misconfiguration that appears everywhere in modern infrastructure and is consistently underestimated: **trusting client-controlled HTTP headers for access control decisions**.

The scenario is common. A public-facing server runs Nginx as a reverse proxy in front of an internal application. The application has an admin panel that should only be accessible from localhost. Someone — reasonably, it seems — decides to enforce this by checking the origin IP in the request. The problem is where they check it.

## The Mistake

```nginx
location /admin {
    if ($http_x_forwarded_for != "127.0.0.1") {
        return 403;
    }
    proxy_pass http://127.0.0.1:8080;
}
```

`$http_x_forwarded_for` is the value of the `X-Forwarded-For` header. That header is set by the **client**. Any external user can send:

```http
GET /admin/diagnostics HTTP/1.1
X-Forwarded-For: 127.0.0.1
```

And the check passes. The admin panel is now accessible from anywhere on the internet.

## Why This Happens

The confusion comes from how `X-Forwarded-For` is supposed to work. In a properly configured stack, the header is set by a trusted upstream proxy — not the client. The internal application reads it to know the real client IP after going through load balancers.

But when Nginx is the **first** point of contact (the internet-facing layer), it is receiving headers from untrusted clients. Trusting those headers for security decisions is equivalent to asking someone for their ID and accepting whatever they write on a piece of paper.

## The Right Approach

**At the Nginx level** — use `$remote_addr` (the actual TCP connection IP) instead of the forwarded header:

```nginx
location /admin {
    allow 127.0.0.1;
    deny all;
    proxy_pass http://127.0.0.1:8080;
}
```

This cannot be forged. The `allow/deny` directives operate on the real network layer.

**At the application level** — never make authorization decisions based on `X-Forwarded-For` alone. If the application needs the real IP, configure Nginx to set a trusted header using `$remote_addr`:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

And strip any client-supplied version of those headers before forwarding:

```nginx
proxy_set_header X-Forwarded-For "";
```

## The Cascade

The reason this is worth writing about is not the misconfiguration itself — it is what came after in a real exercise. The accessible admin panel exposed system diagnostics. The diagnostics exposed log fragments. The log fragments exposed a temporary SSH key path. The key gave access to an internal network segment. The flag was on a host with no public route.

**One bad header. Full internal network access.**

The surface looked small. Two public ports, no obvious login form, no CMS. The weakness was invisible to a passive scan. It only showed up when you paid attention to how the proxy behaved under different inputs.

## Takeaways

- Never use `X-Forwarded-For` or `X-Real-IP` for access control at the edge layer — they are client-controlled
- Restrict sensitive paths using network-layer controls (`allow/deny`, firewall rules), not header checks
- Admin endpoints should not be on the same listener as the public application if at all avoidable
- Verbose diagnostics endpoints are a significant risk — limit what they expose and who can reach them
- A small attack surface in a port scan is not the same as a small attack surface in practice

---

*Related: [Case 01 — Linux Pivot via Reverse Proxy Misconfiguration](https://github.com/juliomauro/sec-writeups/tree/main/cases/01-linux-pivot)*
