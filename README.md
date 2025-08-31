# nginx-cis-documents

# NGINX Hardening and CIS Benchmark Summary

This document provides a summary of NGINX hardening practices based on CIS benchmarks and security best practices.

## Installation & Configuration

### 1. Check NGINX Installation
Verify if NGINX is already installed on your system.

### 2. Install NGINX from Source
Installing from source allows you to include only necessary modules and exclude unwanted ones.
- **Pros**: Customizable module selection
- **Cons**: Requires recompilation for upgrades

### 3. Configure Package Manager Repositories
Ensure your repositories are properly configured to receive the latest security updates.

### 4. Always Use Latest Version
Install the most recent stable version of NGINX to benefit from security patches.

### 5. Install Only Required Modules
- Check loaded modules: `nginx -V`
- Note: NGINX doesn't support module removal; recompilation is required to remove modules

## Module Security

### 6. Remove WebDAV Module
WebDAV enables file operations on your server (create, delete, modify files).
- Module: `http_dav_module`
- Compile without: `--with-http-dav-module`
- Not installed by default in source installations

### 7. Remove Gzip Modules
Compression has been linked to security vulnerabilities like BREACH attack.
- Test: `nginx -V 2>&1 | grep -E '(http_gzip_module|http_gzip_static_module)'`
- Fix: `./configure --without-http_gzip_module --without-http_gzip_static_module`

### 8. Disable Autoindex Module
Disabled by default; ensure it's not enabled in configuration files.
- Test: 
  ```bash
  egrep -i '^\s*autoindex\s+' /etc/nginx/nginx.conf
  egrep -i '^\s*autoindex\s+' /etc/nginx/conf.d/*
  ```

## User & Permission Security

### 9. Run NGINX as Non-Privileged User
Create and use a dedicated non-privileged user (nginx or www-data).
- Check: `passwd -S nginx|www-data`
- Lock: `passwd -l nginx|www-data`

### 10. Remove Home Directory for NGINX User
- Command: `usermod -s /sbin/nologin nginx`

### 11. Secure Configuration Directory
- Set ownership: `chown -R root:root /etc/nginx`
- Set permissions:
  ```bash
  find /etc/nginx -type d -exec chmod go-w {} +
  find /etc/nginx -type f -exec chmod ug-x,o-rwx {} +
  ```
- Test: `find /etc/nginx -type d -exec stat -Lc "%n %a" {} +` (should show 755)

### 12. Protect PID File
- Set root ownership: `stat -L -c "%U:%G" /var/run/nginx.pid`
- Verify permissions

### 13. Secure Core Dump Directory
Core dumps may contain sensitive information.
- Check if enabled: `grep working_directory /etc/nginx/nginx.conf`
- Secure directory:
  ```bash
  chown root:nginx /var/log/nginx
  chmod o-rwx /var/log/nginx
  ```

## Network Security

### 14. Authorized Ports Only
- Verify: `grep -r listen /etc/nginx`

### 15. Reject Unknown Host Names
Prevent access via invalid host headers.
- Test: `curl -k -v https://127.0.0.1 -H 'Host: invalid.host.com'` (should return 404/error)
- Configuration example:
  ```nginx
  server {
      listen 80 default_server;
      listen 443 ssl default_server;
      ssl_certificate /opt/ssl/citydi.com.crt;
      ssl_certificate_key /opt/ssl/citydi.com.key;
      return 404;
  }
  ```

### 16. Set Keepalive Timeout
- Recommended: `keepalive_timeout 10;` (10 seconds or less, not zero)
- Check: `grep -ir keepalive_timeout /etc/nginx`

### 17. Set Send Timeout
- Recommended: `send_timeout 10;` (10 seconds or less, not zero)
- Default: 60s

## Information Disclosure

### 18. Disable Server Tokens
Hide NGINX version information.
- Add to nginx.conf: `server_tokens off;`
- Note: This only hides the version, not the server type

### 19. Customize Error Pages
Edit files in `/usr/share/nginx/html/` to remove NGINX references.

### 20. Disable Hidden File Serving
- Add to configuration:
  ```nginx
  location ~ /\. {
      deny all;
      return 404;
  }
  ```

### 21. Hide Proxy Headers
Prevent information disclosure through response headers.
- Test: `grep proxy_hide_header /etc/nginx/nginx.conf`
- Add to configuration:
  ```nginx
  proxy_hide_header X-Powered-By;
  proxy_hide_header Server;
  ```

## Logging

### 22. Enable Detailed Logging
Custom log format example:
```nginx
log_format main escape=json
'{'
    '"time_local": "$time_local",'
    '"time_iso8601": "$time_iso8601",'
    '"remote_addr": "$remote_addr",'
    '"remote_port": "$remote_port",'
    '"remote_user": "$remote_user",'
    '"request": "$request",'
    '"status": "$status",'
    '"body_bytes_sent": "$body_bytes_sent",'
    '"http_referer": "$http_referer",'
    '"http_user_agent": "$http_user_agent",'
    '"http_host": "$host",'
    '"server_name": "$server_name",'
    '"server_protocol": "$server_protocol",'
    '"server_addr": "$server_addr",'
    '"server_port": "$server_port",'
    '"content_type": "$content_type",'
    '"request_time": "$request_time",'
    '"upstream_addr": "$upstream_addr",'
    '"upstream_status": "$upstream_status",'
    '"upstream_response_time": "$upstream_response_time",'
    '"upstream_connect_time": "$upstream_connect_time"'
'}';
```

### 23. Enable Access Logging
Ensure access logging is enabled.

### 24. Enable Error Logging
Set appropriate log level:
- Example: `error_log /var/log/nginx/error_log.log info;`

### 25. Implement Log Rotation
- Check: 
  ```bash
  cat /etc/logrotate.d/nginx | grep weekly
  cat /etc/logrotate.d/nginx | grep rotate
  ```

### 26. Proxy Pass IP Information
Add to relevant configuration sections:
```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

## Encryption

### 27. Redirect HTTP to HTTPS
```nginx
server {
    listen 80;
    server_name cisecurity.org;
    return 301 https://$host$request_uri;
}
```

### 28. Secure Private Key
- Command: `chmod 400 private.key`

### 29. Use Modern TLS Protocols
Only enable TLSv1.2 and TLSv1.3:
- Web server: `ssl_protocols TLSv1.2 TLSv1.3;`
- Proxy: 
  ```nginx
  location / {
      proxy_pass cisecurity.org;
      proxy_ssl_protocols TLSv1.2 TLSv1.3;
  }
  ```

### 30. Disable Weak Ciphers
- Server block:
  ```nginx
  ssl_ciphers ALL:!EXP:!NULL:!ADH:!LOW:!SSLv2:!SSLv3:!MD5:!RC4;
  ```
- Proxy server:
  ```nginx
  proxy_pass https://cisecurity.org;
  proxy_ssl_ciphers ALL:!EXP:!NULL:!ADH:!LOW:!SSLv2:!SSLv3:!MD5:!RC4;
  ```

### 31. HSTS Preloading (Use with Caution)
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```
**Warning**: This commits your domain to HTTPS-only access in browsers for an extended period.

### 32. Disable Session Resumption
Enable perfect forward secrecy.

### 33. Use HTTP/2.0
Enable HTTP/2 for improved performance and security.

## Access Control

### 34. IP Allow/Deny Filters
```nginx
allow 10.1.1.1;
deny all;
```

### 35. Approved HTTP Methods Only
```nginx
if ($request_method !~ ^(GET|HEAD|POST)$) {
    return 444;
}
```

### 36. Set Client Timeouts
```nginx
client_body_timeout 10;
client_header_timeout 10;
```

### 37. Set Maximum Request Body Size
```nginx
client_max_body_size 100K;
```

### 38. Define Maximum URI Buffer Size
Configure appropriate buffer size limits.

### 39. Limit Connections Per IP
```nginx
limit_conn_zone $binary_remote_addr zone=limitperip:10m;
server {
    limit_conn limitperip 10;
}
```

## Security Headers

### 40. X-Frame-Options Header
```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
```

### 41. X-Content-Type-Options Header
```nginx
add_header X-Content-Type-Options "nosniff" always;
```

### 42. Content Security Policy
```nginx
add_header Content-Security-Policy "default-src 'self'" always;
```

### 43. Referrer Policy
```nginx
add_header Referrer-Policy "no-referrer";
```

## Important Notes

- Test all configuration changes before applying to production
- Regularly update NGINX to address security vulnerabilities
- Monitor logs for suspicious activity
- Consider using security modules like ModSecurity for additional protection
- Regularly review and update your security configuration
