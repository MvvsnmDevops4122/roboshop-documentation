# Frontend

The frontend serves the RoboShop website through Nginx. The website is built with React. The developer runs the build, and the result is plain static files (HTML, JS, CSS, fonts and images), so the server only needs a web server, not Node.js.

Nginx also works as a **reverse proxy**: the browser sends every API call to the frontend (`/api/...`), and Nginx forwards it to the right backend server.

| Browser calls | Nginx forwards to |
|---------------|-------------------|
| `/api/catalogue/...` | `http://<CATALOGUE-IP>:8080/...` |
| `/api/user/...` | `http://<USER-IP>:8080/...` |
| `/api/cart/...` | `http://<CART-IP>:8080/...` |
| `/api/shipping/...` | `http://<SHIPPING-IP>:8080/...` |
| `/api/payment/...` | `http://<PAYMENT-IP>:8080/...` |

> **Check with the developer for the exact version required. This setup uses Nginx 1.26.**

---

## Install Nginx

**You can list modules by using `dnf module list nginx`**

```shell
dnf module disable nginx -y
dnf module enable nginx:1.26 -y
dnf install nginx -y
```

Start & enable the Nginx service:

```shell
systemctl enable nginx
systemctl start nginx
```

**Open `http://<FRONTEND-PUBLIC-IP>` in the browser and make sure you see the default Nginx page.**

---

## Deploy the Website

Remove the default content:

```shell
rm -rf /usr/share/nginx/html/*
```

Download and extract the frontend content:

```shell
curl -L -o /tmp/frontend.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/frontend-v4.zip
cd /usr/share/nginx/html
unzip /tmp/frontend.zip
```

**Open the website again. You should see the RoboShop home page. Products don't load yet, because Nginx doesn't know where the backend servers are.**

---

## Configure the Reverse Proxy

We don't touch the main file `/etc/nginx/nginx.conf`. Its default `server` block already loads every file in `/etc/nginx/default.d/`, so we only add our settings there:

```shell
vim /etc/nginx/default.d/roboshop.conf
```

```nginx
server_tokens off;

# nginx gives every request a unique id ($request_id). It is sent to the
# services and back to the browser, so one click can be traced on all servers
add_header X-Request-Id $request_id always;
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy strict-origin-when-cross-origin always;

gzip on;
gzip_types text/css application/javascript application/json image/svg+xml;
gzip_min_length 1024;

# common settings for all backend calls
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Request-Id      $request_id;
# the services send the id back too, nginx already adds it, so hide theirs
proxy_hide_header X-Request-Id;
proxy_connect_timeout 5s;
proxy_read_timeout    30s;   # 504 if a service takes longer than this

# the trailing / removes the /api/<service> part:
# /api/catalogue/products -> http://<CATALOGUE-IP>:8080/products
location /api/catalogue/ { proxy_pass http://<CATALOGUE-IP>:8080/; }
location /api/user/      { proxy_pass http://<USER-IP>:8080/; }
location /api/cart/      { proxy_pass http://<CART-IP>:8080/; }
location /api/shipping/  { proxy_pass http://<SHIPPING-IP>:8080/; }
location /api/payment/   { proxy_pass http://<PAYMENT-IP>:8080/; }

# when nginx itself cannot reach a service, answer in JSON like the services do.
# errors a service returns on its own (its 404, 503...) pass through untouched
error_page 502 = @bad_gateway;
error_page 504 = @gateway_timeout;
location @bad_gateway {
    default_type application/json;
    return 502 '{"message":"service not reachable (502 Bad Gateway from nginx)"}';
}
location @gateway_timeout {
    default_type application/json;
    return 504 '{"message":"service did not answer in time (504 Gateway Timeout from nginx)"}';
}

# nginx's own connection counters
location = /health {
    stub_status on;
    access_log off;
}

# product images, missing ones fall back to the placeholder
location /images/ {
    expires 1h;
    try_files $uri /images/placeholder.jpg;
}

# file names in /assets contain a hash of their content, so they can be cached forever
location /assets/ {
    expires max;
    try_files $uri =404;
}

# React handles all other paths (/product/RMC, /cart ...) in the browser,
# so every unknown path returns index.html
location / {
    expires -1;
    try_files $uri $uri/ /index.html;
}
```

> **Replace `<CATALOGUE-IP>`, `<USER-IP>`, `<CART-IP>`, `<SHIPPING-IP>` and `<PAYMENT-IP>` with the private IPs of those servers.**

The same content is in [roboshop.conf](roboshop.conf) in this repo.

Check the configuration, then restart Nginx:

```shell
nginx -t
systemctl restart nginx
```

`nginx -t` checks the file for mistakes. Always run it before a restart, so a typo doesn't take the website down.

---

## Verification

```shell
systemctl status nginx
curl http://localhost/health
curl http://localhost/api/catalogue/health
tail -f /var/log/nginx/access.log
```

Open the website and click **System status** at the top. It shows which backend services are up.

| Status from `/api/...` | Meaning |
|------------------------|---------|
| `502` from nginx | Nginx cannot reach that server: wrong IP, service stopped, or port 8080 blocked in the security group |
| `504` from nginx | The service did not answer within 30 seconds |
| `404` with `index.html` | Typo in the `location` path |

> **If every `/api/` call gives 502 and the IPs are correct:** SELinux may be blocking Nginx from making network connections. Check with `getenforce`. If it says `Enforcing`, allow it:
>
> ```shell
> setsebool -P httpd_can_network_connect 1
> ```

---

## Security Group

Allow **inbound TCP 80** from `0.0.0.0/0` (the internet).
