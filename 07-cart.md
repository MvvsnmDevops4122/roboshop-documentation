# Cart

Cart is a microservice that keeps each shopper's cart in Redis. When an item is added, Cart asks Catalogue for the current price and stock, so a cart never holds more than what is in stock.

> **Developer has chosen Node.js. Check with the developer which version is needed. This setup requires Node.js >= 24.**

---

## Install Node.js

**You can list modules by using `dnf module list nodejs`**

```shell
dnf module disable nodejs -y
dnf module enable nodejs:24 -y
dnf install nodejs -y
```

---

## Create Application User

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

---

## Download the Application

```shell
mkdir /app
curl -L -o /tmp/cart.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/cart-v4.zip
cd /app
unzip /tmp/cart.zip
```

---

## Install Dependencies

```shell
cd /app
npm install
```

---

## Configure SystemD Service

```shell
vim /etc/systemd/system/cart.service
```

```ini
[Unit]
Description=Cart Service
After=network-online.target

[Service]
User=roboshop
Environment=REDIS_HOST=<REDIS-SERVER-IPADDRESS>
Environment=CATALOGUE_HOST=<CATALOGUE-SERVER-IPADDRESS>
Environment=CATALOGUE_PORT=8080
ExecStart=/bin/node /app/server.js
SyslogIdentifier=cart
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<REDIS-SERVER-IPADDRESS>` and `<CATALOGUE-SERVER-IPADDRESS>` with the private IPs of those servers.**

Optional: `CART_TTL_SECONDS` (default `3600`) is how long an unchanged cart is kept in Redis.

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable cart
systemctl start cart
```

---

## Verification

```shell
systemctl status cart
journalctl -u cart -f
curl http://localhost:8080/health
```

`/health` returns `{"app":"OK","redis":true}`.

Add a product to a test cart. This also checks the connection to Catalogue:

```shell
curl -i -X POST http://localhost:8080/add/test/RMC/1
curl http://localhost:8080/cart/test
```

| Response | Meaning |
|----------|---------|
| `200` | Works |
| `404` | Product not found |
| `409` | Not enough stock |
| `502` | Cart cannot reach Catalogue (check the IP, the Catalogue service and the security group) |
| `504` | Catalogue did not answer within 5 seconds |
| `503` | Redis is down |

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | 200, or 503 if Redis is down |
| GET | `/cart/<id>` | the cart, 404 if empty |
| DELETE | `/cart/<id>` | empty the cart, used by payment |
| POST | `/add/<id>/<sku>/<qty>` | add items |
| POST | `/update/<id>/<sku>/<qty>` | change quantity, `0` removes the item |
| GET | `/rename/<from>/<to>` | move the guest cart to the user at login |
| POST | `/shipping/<id>` | add the shipping cost, used by shipping |
| GET | `/metrics` | Prometheus metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend`, `roboshop-shipping` and `roboshop-payment`.
