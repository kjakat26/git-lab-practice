# NGINX CONFIGURATION & LOAD BALANCING - TECHNICAL INTERVIEW DEEP DIVE

## PART 1: NGINX BASICS

**Q: What is Nginx and when would you use it instead of AWS/Azure load balancers?**

Nginx is a high-performance reverse proxy and load balancer. It's open-source and very flexible. You use it when you need:

1. **Fine-grained control**: AWS ELB and Azure LB are managed services—you can't customize behavior at the level Nginx allows.

2. **On-premises deployments**: You can't use AWS ELB on your own servers. Nginx runs on any Linux server.

3. **Complex routing logic**: Route based on URL patterns, hostnames, headers, or custom logic that managed LBs can't do.

4. **Cost optimization**: Nginx is free (open-source). You pay for servers, not per-request fees like AWS.

5. **Advanced caching**: Nginx can cache responses, reducing load on backends.

**At Globe Telecom**, we used Nginx because we needed:
- Telecom-specific traffic shaping rules
- Custom logging for compliance
- Fine-grained control over failover behavior
- Cost control—we didn't want AWS billing surprises

**Think of it this way**: AWS ELB is like a managed car rental service—convenient, reliable, but you can't modify the car. Nginx is like owning your own car—more work to maintain, but total control.

---

## PART 2: NGINX CONFIGURATION FILE STRUCTURE

**Q: Walk me through a basic Nginx configuration file.**

The Nginx config is hierarchical. Here's the basic structure:

```nginx
# Main context - applies globally
user nginx;
worker_processes auto;  # Use all CPU cores
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 10000;  # Max connections per worker
    use epoll;  # High-performance event model (Linux)
}

http {
    # Global HTTP settings
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
    
    access_log /var/log/nginx/access.log main;
    
    # Performance tuning
    sendfile on;  # Use zero-copy file transmission
    tcp_nopush on;  # Send complete packets at once
    tcp_nodelay on;  # Don't wait for full packets (reduces latency)
    keepalive_timeout 65;  # Keep connections alive for reuse
    
    # Upstream backend servers
    upstream backend_pool {
        server backend1.example.com:8080 weight=5;
        server backend2.example.com:8080 weight=3;
        server backend3.example.com:8080 backup;
    }
    
    # Virtual host / server block
    server {
        listen 80;
        listen [::]:80;
        server_name example.com www.example.com;
        
        return 301 https://$server_name$request_uri;
    }
    
    # HTTPS server
    server {
        listen 443 ssl;
        listen [::]:443 ssl;
        server_name example.com www.example.com;
        
        ssl_certificate /etc/nginx/certs/certificate.crt;
        ssl_certificate_key /etc/nginx/certs/private.key;
        
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        
        location / {
            proxy_pass http://backend_pool;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }
        
        location /api/ {
            proxy_pass http://backend_pool;
            proxy_cache_valid 200 1m;
            proxy_cache_key "$scheme$request_method$host$request_uri";
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

**Key concepts:**
- **Upstream block**: Defines backend servers
- **Server block**: Defines a virtual host
- **Location block**: Handles specific URL paths
- **Proxy directives**: Control how requests are forwarded

---

## PART 3: LOAD BALANCING ALGORITHMS

**Q: What load balancing algorithms does Nginx support, and when would you use each?**

Nginx supports several algorithms:

### 1. Round Robin (default):
```nginx
upstream backend_pool {
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}
```
**Use when**: All backends have similar capacity.

### 2. Weighted Round Robin:
```nginx
upstream backend_pool {
    server backend1.example.com:8080 weight=5;
    server backend2.example.com:8080 weight=3;
    server backend3.example.com:8080 weight=1;
}
```
**Use when**: Backends have different capacities.
**Example**: If backend1 is 32-CPU and backend2 is 8-CPU, use weight=4:1.

### 3. Least Connections:
```nginx
upstream backend_pool {
    least_conn;
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}
```
**Use when**: Connections are long-lived (WebSockets, persistent connections).
**How it works**: Routes new connections to the server with fewest active connections.
**Example**: A gaming platform with persistent player connections—you want to balance active players, not just round-robin.

### 4. IP Hash:
```nginx
upstream backend_pool {
    ip_hash;
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}
```
**Use when**: You need session persistence (same client always hits same backend).
**Problem**: If a backend goes down, all its clients get rehashed to other backends (session loss).

### 5. Consistent Hash:
```nginx
upstream backend_pool {
    hash $request_uri consistent;
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}
```
**Use when**: You want consistent hashing (minimal disruption if backends change).

**For Pinnacle gaming platform**: Use least_conn for player WebSocket connections and consistent hash for betting APIs.

---

## PART 4: HEALTH CHECKS & FAILOVER

**Q: How do you configure health checks in Nginx so unhealthy backends are removed?**

Nginx has limited built-in health checking. Here's the approach:

### Passive Health Checks (built-in):
```nginx
upstream backend_pool {
    server backend1.example.com:8080 max_fails=3 fail_timeout=10s;
    server backend2.example.com:8080 max_fails=3 fail_timeout=10s;
    server backend3.example.com:8080 backup;
}

server {
    location / {
        proxy_pass http://backend_pool;
        
        # If a request fails 3 times, mark server down for 10s
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 2;  # Try up to 2 backends before failing
    }
}
```

**How it works**: Nginx watches for failures (connection refused, timeout, 5xx errors). After 3 failures, marks server down for 10 seconds.

**Problem**: Passive—it detects failures only after clients hit the backend.

### For Production:
Since Nginx open-source doesn't have active health checks, use an external script that runs every 5 seconds and monitors backends.

**At Globe Telecom**, we used this approach with a health check script that updated Nginx upstream configuration dynamically.

---

## PART 5: HIGH AVAILABILITY NGINX SETUP

**Q: How would you set up Nginx for high availability?**

Use active-active HA with Keepalived:

### Nginx Server 1 & 2 (identical config):
```nginx
upstream backend_pool {
    server backend1:8080;
    server backend2:8080;
    server backend3:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Keepalived Configuration (Master):
```
global_defs {
    router_id NGINX_HA
}

vrrp_script check_nginx {
    script "/usr/local/bin/check_nginx.sh"
    interval 2
    weight -20
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass secret123
    }
    
    virtual_ipaddress {
        10.0.1.5/24  # Virtual IP that clients use
    }
    
    track_script {
        check_nginx
    }
}
```

**How it works:**
1. Both Nginx servers have identical configs
2. Master broadcasts heartbeats saying "I'm alive"
3. If no heartbeat for 3 seconds, standby becomes master
4. Virtual IP (10.0.1.5) moves to new master
5. Clients always hit 10.0.1.5, unaware of failover

**Failover time**: 2-3 seconds.

---

## PART 6: SSL/TLS TERMINATION

**Q: Walk me through how you'd configure SSL/TLS termination in Nginx. Security best practices?**

SSL termination means Nginx handles HTTPS from clients, then proxies to backends.

### Basic SSL Configuration:
```nginx
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/nginx/certs/example.com.crt;
    ssl_certificate_key /etc/nginx/certs/example.com.key;
    ssl_trusted_certificate /etc/nginx/certs/ca_bundle.crt;
    
    # Modern TLS only
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Secure ciphers
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
    ssl_prefer_server_ciphers on;
    
    # Session settings
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    
    # HSTS - force HTTPS for future requests
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    # OCSP stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4;
    
    location / {
        proxy_pass http://backend_pool;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### Security Best Practices:

1. **Certificate Management**:
   - Use Let's Encrypt for free, auto-renewing certs
   - Store private keys with 600 permissions
   - Auto-renewal via cron job

2. **TLS Versions**: Only TLSv1.2+

3. **Cipher Selection**: Use ECDHE for forward secrecy, avoid weak ciphers

4. **HSTS Header**: Forces browsers to use HTTPS

5. **OCSP Stapling**: Pre-fetch certificate status (faster)

**For Pinnacle gaming platform**:
- Use TLSv1.3 (fast, secure)
- Strong ciphers (customer data includes payment info)
- HSTS enabled
- Auto-renewal (certificate expiration = outage)

---

## PART 7: CACHING & PERFORMANCE

**Q: How do you configure caching in Nginx?**

Nginx caching reduces backend load. But be careful—cache invalid data and users lose money.

### Basic Caching Configuration:
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=1g inactive=60m;

server {
    location / {
        proxy_pass http://backend_pool;
        
        proxy_cache my_cache;
        proxy_cache_valid 200 1m;
        proxy_cache_valid 404 1m;
        proxy_cache_valid 500 502 503 504 1s;
        
        proxy_cache_key "$scheme$request_method$host$request_uri$args";
        
        add_header X-Cache-Status $upstream_cache_status;
        
        # Skip cache for authenticated requests
        set $skip_cache 0;
        if ($http_authorization) {
            set $skip_cache 1;
        }
        proxy_cache_bypass $skip_cache;
        proxy_no_cache $skip_cache;
    }
}
```

### What to Cache:

✅ **Cache these**:
- Static assets (images, CSS, JS) — long TTL (1 hour+)
- API responses that don't change (odds, schedules) — medium TTL (5-15 min)
- User profile data — medium TTL (if stale data acceptable)

❌ **Don't cache**:
- Account balance (must be real-time)
- Active bets (must be real-time)
- Payment transactions (never)
- Responses with Set-Cookie header

**For Pinnacle**:
- Cache odds/lines (update every 5 min)
- Cache sports data (teams, schedules) — long TTL
- Do NOT cache account balance or active bets

---

## PART 8: RATE LIMITING

**Q: How would you rate-limit requests in Nginx?**

Rate limiting protects backend from overload.

### Basic Rate Limiting:
```nginx
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=strict:10m rate=1r/s;

server {
    location /api/ {
        limit_req zone=general burst=20 nodelay;
        proxy_pass http://backend_pool;
    }
    
    location /api/withdrawals {
        limit_req zone=strict burst=5 nodelay;
        proxy_pass http://backend_pool;
    }
}
```

### Rate Limiting by User ID:
```nginx
map $http_authorization $user_id {
    default $binary_remote_addr;
    ~*"Basic (?<auth>.+)" $auth;
}

limit_req_zone $user_id zone=user_limit:10m rate=5r/s;

location /api/ {
    limit_req zone=user_limit burst=10 nodelay;
    proxy_pass http://backend_pool;
}
```

### Handling Rate Limited Requests:
```nginx
location /api/ {
    limit_req zone=general burst=20 nodelay;
    proxy_pass http://backend_pool;
    
    error_page 429 = @rate_limited;
}

location @rate_limited {
    return 429 'Too many requests. Try again later.';
    add_header Retry-After 60;
}
```

**For Pinnacle**:
- Strict on betting endpoints (prevent automated betting)
- Moderate on API queries
- Different limits for VIP users

---

## PART 9: TROUBLESHOOTING

**Q: Nginx is slow. Walk me through your troubleshooting process.**

### Systematic approach:

**Step 1: Check Nginx is Running**
```bash
$ sudo systemctl status nginx
$ sudo nginx -t
$ curl http://localhost/health
```

**Step 2: Check Resource Usage**
```bash
$ top -p $(pgrep -f 'nginx: master')
$ free -h
$ df -h /var/cache/nginx
$ netstat -an | grep ESTABLISHED | wc -l
```

**Step 3: Check Logs**
```bash
$ tail -100 /var/log/nginx/error.log
$ tail -100 /var/log/nginx/access.log
```

**Step 4: Monitor Request Flow**
```bash
$ sudo ss -tonp | grep nginx
$ grep upstream /var/log/nginx/access.log | awk '{print $NF}' | sort | uniq -c
```

**Step 5: Test Backend Directly**
```bash
$ curl -v http://backend1:8080/
$ time curl http://backend1:8080/
```

**Step 6: Check Nginx Config**
```bash
$ grep "worker_connections" /etc/nginx/nginx.conf
$ grep "upstream_conn_timeout\|proxy_connect_timeout" /etc/nginx/nginx.conf
```

### Real Example:

At **Globe Telecom**, we had slow Nginx:

```bash
$ tail /var/log/nginx/error.log
[error] socket() failed (24: Too many open files)

# Problem: File descriptor limit too low
$ ulimit -n
1024

# Fix in /etc/security/limits.conf:
nginx soft nofile 65536
nginx hard nofile 65536

$ sudo systemctl restart nginx
# Performance improved immediately
```

---

## PART 10: INTERVIEW DEEP DIVES

### Q1: Session Persistence in Nginx

**A**: Depends on architecture:

**Option 1 - Sticky Sessions (IP Hash)**:
```nginx
upstream backend_pool {
    ip_hash;
    server backend1:8080;
    server backend2:8080;
}
```
Problem: If backend dies, sessions are lost.

**Option 2 - Shared Session Store (Recommended)**:
```
# Store sessions in Redis
# Any backend can serve any client
# Nginx with least_conn (no sticky required)
```

**Option 3 - JWT Tokens**:
```
# Client includes JWT in every request
# Any backend can verify
# Stateless architecture
```

**For Pinnacle**: Use shared session store (Redis) + least_conn balancing.

---

### Q2: Zero-Downtime Deployments

**A**: Use graceful reload:

```bash
$ sudo nginx -s reload
# Master loads new config
# New workers spawned
# Old workers drain connections gracefully
# Zero connection loss
```

For backend deployments:
```
# 1. Mark backend1 as down
upstream backend_pool {
    server backend1:8080 down;
    server backend2:8080;
    server backend3:8080;
}
# 2. Deploy to backend1
# 3. Enable backend1
# 4. Repeat for backend2, backend3
# Result: Zero downtime
```

---

### Q3: Debugging Timeouts

**A**:
```bash
# Check which timeout is too short
proxy_connect_timeout 5s;
proxy_read_timeout 30s;
proxy_send_timeout 30s;

# Check if backend is busy
netstat -an | grep WAIT | wc -l

# Monitor real-time
watch -n 1 'tail -1 /var/log/nginx/access.log'
```

---

## INTERVIEW TIPS

1. **Show you understand the OSI model** (L4 vs L7)
2. **Use concrete examples** from your work
3. **Understand tradeoffs** (sticky vs shared store, caching TTL, etc.)
4. **Know what happens when components fail**
5. **Admit unknowns gracefully** ("I haven't used Nginx Plus, but...")
6. **Defend your decisions** with reasoning

Good luck! 🚀
