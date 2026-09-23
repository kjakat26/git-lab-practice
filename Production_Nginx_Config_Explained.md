# PRODUCTION NGINX CONFIGURATION - ENTERPRISE SETUP WITH 3 BACKEND SERVERS
# For: Pinnacle Sports Betting Platform
# This is a real-world production configuration with detailed explanations

# ============================================================================
# MAIN CONTEXT - GLOBAL NGINX SETTINGS
# ============================================================================

user nginx;  
# Run Nginx worker processes as 'nginx' user (not root for security)
# This user should have no shell access and minimal permissions

worker_processes auto;
# Automatically detect number of CPU cores and spawn that many workers
# For 4-core server: spawns 4 worker processes
# Each worker handles requests independently, load is distributed

worker_rlimit_nofile 65535;
# Maximum number of open file descriptors per worker
# Important for high-traffic servers (default 1024 is too low)
# Each connection needs 1 file descriptor, so this allows ~65k connections per worker

error_log /var/log/nginx/error.log warn;
# Log file for Nginx errors (config errors, worker crashes, etc.)
# 'warn' level means only warnings and errors (not info/debug)
# Levels: debug, info, notice, warn, error, crit, alert, emerg

pid /var/run/nginx.pid;
# Process ID file - used by systemctl to manage Nginx

# ============================================================================
# EVENTS CONTEXT - CONNECTION PROCESSING
# ============================================================================

events {
    worker_connections 10000;
    # Maximum number of simultaneous connections per worker
    # Total capacity = worker_processes * worker_connections
    # Example: 4 workers * 10000 = 40,000 concurrent connections
    # For betting platform: need high concurrency (many players betting simultaneously)
    
    use epoll;
    # Event processing mechanism (Linux only, highly efficient)
    # epoll = efficient polling mechanism for handling many connections
    # Without this: uses select() which is slower for 1000+ connections
    # Other options: select, poll, kqueue (BSD), ioctl (Windows)
    
    multi_accept on;
    # Accept multiple connections in one worker cycle
    # Improves throughput by processing multiple connections per loop iteration
    # Can increase latency slightly but better overall throughput for high traffic
}

# ============================================================================
# HTTP CONTEXT - GLOBAL HTTP SETTINGS
# ============================================================================

http {
    # Include MIME types (so .jpg returns image/jpeg, .css returns text/css, etc.)
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    # If MIME type unknown, treat as binary file (download it)
    
    # ========================================================================
    # LOGGING CONFIGURATION
    # ========================================================================
    
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';
    # Log format with timing information
    # $remote_addr = client IP
    # $request = GET /api/bets HTTP/1.1
    # $status = HTTP status (200, 502, etc.)
    # $body_bytes_sent = response size
    # $request_time = total time to process request
    # $upstream_response_time = time backend took to respond
    # This lets us identify slow backends and slow requests
    
    log_format cache_status '$remote_addr - [$time_local] '
                           '"$request" $status $upstream_cache_status '
                           '(size: $body_bytes_sent)';
    # Separate log format for cache hits/misses
    # $upstream_cache_status = HIT/MISS/EXPIRED/BYPASS
    
    access_log /var/log/nginx/access.log main buffer=32k flush=5s;
    # Buffer log writes (32KB buffer) and flush every 5 seconds
    # This reduces disk I/O significantly (instead of writing every request)
    # Without buffering: massive I/O overhead on high-traffic servers
    
    # ========================================================================
    # PERFORMANCE TUNING
    # ========================================================================
    
    sendfile on;
    # Use kernel's sendfile() system call instead of read() + write()
    # Kernel copies file directly from disk to network socket
    # Much faster than user-space copying (saves context switches)
    
    tcp_nopush on;
    # Don't send partial TCP packets
    # Accumulate data and send complete packets (better network efficiency)
    # Reduces overhead and increases throughput
    
    tcp_nodelay on;
    # Send data immediately (don't wait for full packets)
    # Good for low-latency applications (like betting platform)
    # Reduces latency but increases packet count
    # Note: tcp_nopush and tcp_nodelay together = send full packets ASAP
    
    keepalive_timeout 65 60;
    # Keep connections alive for 65 seconds
    # Send "Connection: close" header after 60 seconds to clients
    # Connection pooling: reuse connections for multiple requests (faster)
    # Example: player places 10 bets = 1 connection, 10 requests
    
    client_max_body_size 10m;
    # Maximum request body size (POST data)
    # Betting platform probably doesn't need huge bodies, but set reasonable limit
    # Prevents memory exhaustion from huge requests
    
    gzip on;
    # Enable compression for responses
    gzip_min_length 1000;
    # Only compress responses larger than 1KB (overhead not worth it for tiny responses)
    gzip_types text/plain text/css text/javascript application/json application/javascript;
    # Compress these content types (images already compressed, skip them)
    gzip_vary on;
    # Add Vary: Accept-Encoding header (for caches to work correctly)
    
    # ========================================================================
    # UPSTREAM BACKEND SERVERS
    # ========================================================================
    
    upstream betting_api_backend {
        # Connection pooling: reuse connections to backends (faster)
        keepalive 32;  # Keep 32 idle connections to each backend
        
        # Production backend servers (3 servers for redundancy)
        server 10.0.1.100:8080 weight=1 max_fails=3 fail_timeout=10s;
        # weight=1: equal load distribution (1:1:1 ratio)
        # max_fails=3: mark server down after 3 failed requests
        # fail_timeout=10s: wait 10 seconds before retrying a failed server
        # Production IP (internal only, not public)
        
        server 10.0.1.101:8080 weight=1 max_fails=3 fail_timeout=10s;
        # Second backend server (identical config)
        # These could be in same datacenter or different AZs
        
        server 10.0.1.102:8080 weight=1 max_fails=3 fail_timeout=10s;
        # Third backend server (identical config)
        # 3 servers = if 1 fails, still have 50% capacity
        
        # Least connections: route to server with fewest active connections
        # Good for long-lived connections (WebSockets, betting streams)
        least_conn;
    }
    
    upstream static_backend {
        # For serving static files (images, CSS, JS)
        # Could point to CDN or static file server
        keepalive 16;
        
        server 10.0.1.110:8080 weight=1;
        server 10.0.1.111:8080 weight=1;
        # Fewer backends for static files (less traffic than APIs)
    }
    
    # ========================================================================
    # RATE LIMITING ZONES
    # ========================================================================
    
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
    # Zone name: api_limit
    # Shared memory: 10MB (enough for 160,000 IPs at 64 bytes per entry)
    # Rate: 100 requests per second per IP
    # $binary_remote_addr = use client IP as key (more efficient than $remote_addr)
    
    limit_req_zone $binary_remote_addr zone=betting_limit:10m rate=10r/s;
    # Stricter limit for betting endpoint (prevent automated betting)
    # 10 req/s per IP = 600 bets/minute (reasonable for one player)
    
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    # Connection limit zone (not just requests, but open connections)
    limit_conn addr 100;
    # Max 100 simultaneous connections per IP
    # Prevents connection exhaustion attacks
    
    # ========================================================================
    # CACHING ZONES
    # ========================================================================
    
    proxy_cache_path /var/cache/nginx/api 
                     levels=1:2 
                     keys_zone=api_cache:10m 
                     max_size=1g 
                     inactive=60m
                     use_temp_path=off;
    # Cache configuration
    # /var/cache/nginx/api = physical cache directory
    # levels=1:2 = directory structure /a/bb/xxx (more efficient than single dir)
    # keys_zone=api_cache:10m = named zone, 10MB shared memory for cache keys
    # max_size=1g = maximum cache size (1GB)
    # inactive=60m = delete cache entries not accessed for 60 minutes
    # use_temp_path=off = write directly to cache (instead of temp first)
    
    proxy_cache_path /var/cache/nginx/static
                     levels=1:2
                     keys_zone=static_cache:20m
                     max_size=5g
                     inactive=180m
                     use_temp_path=off;
    # Separate cache for static files (larger, longer TTL)
    
    # ========================================================================
    # HTTP/2 PUSH CONFIGURATION
    # ========================================================================
    
    # Enable server push for critical CSS/JS (optional, advanced)
    # http2_push_preload on;
    
    # ========================================================================
    # HTTP REDIRECT SERVER (HTTP -> HTTPS)
    # ========================================================================
    
    server {
        listen 80;
        listen [::]:80;  # IPv6 support
        server_name pinnacle.example.com;
        
        # Redirect all HTTP traffic to HTTPS
        # Ensures all connections are encrypted
        return 301 https://$server_name$request_uri;
        # $server_name = preserve original hostname
        # $request_uri = preserve path and query string
        # Example: http://pinnacle.example.com/api/bets?user=123
        #       -> https://pinnacle.example.com/api/bets?user=123
    }
    
    # ========================================================================
    # MAIN HTTPS SERVER - PRODUCTION
    # ========================================================================
    
    server {
        listen 443 ssl http2 deferred;
        # Port 443 = HTTPS
        # ssl = enable SSL/TLS
        # http2 = HTTP/2 protocol (faster than HTTP/1.1)
        # deferred = accept connections faster (small optimization)
        
        listen [::]:443 ssl http2 deferred;
        # IPv6 support
        
        server_name pinnacle.example.com;
        # Domain name for this server
        
        # ====================================================================
        # SSL/TLS CONFIGURATION
        # ====================================================================
        
        ssl_certificate /etc/nginx/certs/pinnacle.example.com.crt;
        # Public certificate (from Let's Encrypt or CA)
        
        ssl_certificate_key /etc/nginx/certs/pinnacle.example.com.key;
        # Private key (keep this secure! chmod 600)
        
        ssl_trusted_certificate /etc/nginx/certs/ca_bundle.crt;
        # Intermediate + root certificates (for OCSP stapling)
        
        ssl_protocols TLSv1.2 TLSv1.3;
        # Only modern TLS versions (TLSv1.0/1.1 deprecated)
        # TLSv1.3 is faster and more secure
        
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384';
        # Strong cipher suites only
        # ECDHE = forward secrecy (past keys not compromised if private key stolen)
        # AES128/256 = encryption algorithm
        # GCM = authenticated encryption (prevents tampering)
        
        ssl_prefer_server_ciphers on;
        # Use server's cipher list (not client's) - server knows best
        
        ssl_session_cache shared:SSL:50m;
        # Cache TLS sessions in shared memory (50MB)
        # New connections can resume sessions (faster handshake)
        
        ssl_session_timeout 1d;
        # Cached session valid for 1 day
        
        ssl_session_tickets off;
        # Disable TLS session tickets (security consideration)
        # Some versions have vulnerabilities
        
        # OCSP Stapling - server fetches certificate revocation status
        # Client gets it with certificate (faster, more private)
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /etc/nginx/certs/ca_bundle.crt;
        resolver 8.8.8.8 8.8.4.4 valid=300s;
        # Resolver IPs (Google DNS) for OCSP lookups
        
        # ====================================================================
        # SECURITY HEADERS
        # ====================================================================
        
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
        # HSTS: Force browsers to use HTTPS for 1 year
        # Prevents man-in-the-middle attacks
        
        add_header X-Content-Type-Options "nosniff" always;
        # Prevent browser from guessing content type
        # Example: fake .jpg that's actually executable
        
        add_header X-Frame-Options "SAMEORIGIN" always;
        # Prevent clickjacking (site can't be embedded in iframe)
        
        add_header X-XSS-Protection "1; mode=block" always;
        # Enable browser XSS protection (legacy, but good fallback)
        
        add_header Referrer-Policy "strict-origin-when-cross-origin" always;
        # Don't send referrer to other domains
        
        add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
        # Disable browser permissions that we don't use
        
        # ====================================================================
        # LOGGING
        # ====================================================================
        
        access_log /var/log/nginx/pinnacle.access.log main buffer=32k flush=5s;
        error_log /var/log/nginx/pinnacle.error.log warn;
        
        # ====================================================================
        # STATIC FILES - IMAGES, CSS, JS, FONTS
        # ====================================================================
        
        location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
            # Regex: match files ending with these extensions
            # ~* = case-insensitive regex
            
            proxy_pass http://static_backend;
            # Route to static backend servers
            
            # Cache static files for 1 year (very long)
            # Browsers won't request again unless cache cleared
            proxy_cache static_cache;
            proxy_cache_valid 200 365d;
            proxy_cache_key "$scheme$proxy_host$request_uri";
            
            # Add cache status header (debug info)
            add_header X-Cache-Status $upstream_cache_status;
            
            # Logging for static files (optional - reduces log noise)
            access_log /var/log/nginx/static.access.log main buffer=32k flush=5s;
        }
        
        # ====================================================================
        # HEALTH CHECK ENDPOINT
        # ====================================================================
        
        location /health {
            # Simple endpoint for load balancers/monitoring
            access_log off;  # Don't log health checks (reduces noise)
            return 200 '{"status":"ok"}';
            add_header Content-Type application/json;
        }
        
        location /readiness {
            # Kubernetes readiness probe endpoint
            access_log off;
            proxy_pass http://betting_api_backend;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            
            # If any backend responds, we're ready
            # If all fail, return 503
        }
        
        # ====================================================================
        # API ENDPOINTS - BETTING SYSTEM
        # ====================================================================
        
        location /api/ {
            # All API endpoints
            
            proxy_pass http://betting_api_backend;
            # Route to upstream backend pool
            
            # ================================================================
            # PASS HEADERS TO BACKEND
            # ================================================================
            
            proxy_set_header Host $host;
            # Pass original host header (not proxy's hostname)
            # Backend needs to know: was request for pinnacle.com or api.pinnacle.com?
            
            proxy_set_header X-Real-IP $remote_addr;
            # Pass client's real IP
            # Otherwise backend sees Nginx's IP and can't do IP-based checks
            
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            # Pass chain of IPs (client, proxy1, proxy2, etc.)
            # Example: X-Forwarded-For: 1.2.3.4, 5.6.7.8 (if multiple proxies)
            
            proxy_set_header X-Forwarded-Proto $scheme;
            # Pass protocol (http or https)
            # Backend needs to know: was original request HTTPS?
            # Some apps redirect HTTPS URLs differently
            
            proxy_set_header X-Forwarded-Host $server_name;
            # Pass original hostname (if proxying across domains)
            
            # ================================================================
            # TIMEOUTS
            # ================================================================
            
            proxy_connect_timeout 5s;
            # Max 5 seconds to establish connection to backend
            # If backend is completely down, fail fast
            
            proxy_send_timeout 30s;
            # Max 30 seconds to send request to backend
            # For normal API requests, this is plenty
            
            proxy_read_timeout 60s;
            # Max 60 seconds to read response from backend
            # Some betting/odds calculations might take time
            # If longer, increase (or implement async)
            
            # ================================================================
            # BUFFERING
            # ================================================================
            
            proxy_buffering on;
            # Buffer responses from backend before sending to client
            # Faster: backend can finish and close connection
            # Without buffering: Nginx streams response (slow backend blocks)
            
            proxy_buffer_size 4k;
            # Buffer size for headers (usually 4k is enough)
            
            proxy_buffers 8 4k;
            # 8 buffers of 4k each = 32k total buffer
            # If response > 32k, write to disk (temp file)
            
            proxy_busy_buffers_size 8k;
            # When 8k data ready, start sending to client
            # Balances buffering vs responsiveness
            
            # ================================================================
            # CONNECTION POOLING
            # ================================================================
            
            proxy_http_version 1.1;
            # Use HTTP/1.1 for connection reuse
            # HTTP/1.0 closes connection after each request
            
            proxy_set_header Connection "";
            # Don't close connection (reuse it)
            # Upstream block has keepalive 32 to maintain pool
            
            # ================================================================
            # CACHING API RESPONSES (selective)
            # ================================================================
            
            proxy_cache api_cache;
            proxy_cache_valid 200 5m;  # Cache successful responses for 5 minutes
            proxy_cache_valid 404 1m;  # Cache 404s for 1 minute
            proxy_cache_valid 500 502 503 504 1s;  # Cache errors briefly
            
            proxy_cache_key "$scheme$request_method$host$request_uri$http_authorization";
            # Cache key: must include auth to avoid serving cached data for different users
            
            # ================================================================
            # BYPASS CACHE FOR CERTAIN REQUESTS
            # ================================================================
            
            # Don't cache POST/PUT/DELETE (only GET)
            proxy_cache_methods GET HEAD;
            
            # Don't cache if Authorization header present (user-specific data)
            proxy_no_cache $http_authorization;
            proxy_cache_bypass $http_authorization;
            
            # Don't cache if ?nocache query param present
            set $skip_cache 0;
            if ($args ~* "nocache") {
                set $skip_cache 1;
            }
            proxy_cache_bypass $skip_cache;
            proxy_no_cache $skip_cache;
            
            # ================================================================
            # RATE LIMITING
            # ================================================================
            
            limit_req zone=api_limit burst=50 nodelay;
            # Max 100 req/s, allow burst of 50 extra
            # nodelay = reject immediately instead of queueing
            
            limit_conn addr 100;
            # Max 100 simultaneous connections per IP
            
            # ================================================================
            # ERROR HANDLING
            # ================================================================
            
            proxy_intercept_errors on;
            # Intercept 4xx/5xx from backend, show custom error page
            
            error_page 502 503 504 /maintenance.html;
            # If backend down, show maintenance page
        }
        
        # ====================================================================
        # BETTING ENDPOINT - STRICTER RATE LIMITING
        # ====================================================================
        
        location /api/v1/bets/place {
            # Betting is critical, need stricter protection
            
            limit_req zone=betting_limit burst=5 nodelay;
            # Only 10 req/s per IP (not 100)
            # Prevents automated betting scripts
            
            proxy_pass http://betting_api_backend;
            
            # All the proxy settings from /api/ above...
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_connect_timeout 5s;
            proxy_send_timeout 30s;
            proxy_read_timeout 60s;
            
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            
            # No caching for bets (every request must hit backend)
            proxy_cache off;
            proxy_no_cache 1;
        }
        
        # ====================================================================
        # CATCH-ALL - ROOT LOCATION
        # ====================================================================
        
        location / {
            # Frontend (React, Vue, etc.) or default handling
            
            proxy_pass http://betting_api_backend;
            
            # Standard proxy headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 30s;
            proxy_read_timeout 60s;
        }
    }
}

# ============================================================================
# END OF CONFIGURATION
# ============================================================================

# HOW TO USE THIS CONFIG:
# 1. Save to /etc/nginx/nginx.conf (or /etc/nginx/sites-available/pinnacle.conf)
# 2. Create directories:
#    sudo mkdir -p /var/cache/nginx/{api,static}
#    sudo chown -R nginx:nginx /var/cache/nginx
# 3. Test config:
#    sudo nginx -t
# 4. Reload Nginx:
#    sudo systemctl reload nginx
# 5. Monitor logs:
#    tail -f /var/log/nginx/pinnacle.access.log
#    tail -f /var/log/nginx/pinnacle.error.log

# MONITORING:
# Check active connections: sudo ss -tonp | grep nginx
# Check cache hit rate: grep HIT /var/log/nginx/pinnacle.access.log | wc -l
# Check backend health: curl -v http://10.0.1.100:8080/health

# PERFORMANCE TUNING:
# If slow: increase worker_processes, worker_connections, buffer sizes
# If high CPU: reduce buffer sizes, disable caching, lower rate limits
# If high memory: reduce cache sizes, reduce keepalive connections
# If timeouts: increase proxy_read_timeout, check backend health

# SECURITY CHECKLIST:
# [ ] Update server IPs and hostnames
# [ ] Install valid SSL certificates (not self-signed in production)
# [ ] Set proper file permissions (chmod 600 on private key)
# [ ] Configure firewall rules
# [ ] Set up log rotation (logrotate)
# [ ] Monitor error logs for attacks
# [ ] Regular security updates (nginx, OS, packages)
