# User

User is a microservice that handles login, registration and order history in RoboShop. Passwords are stored as bcrypt hashes, never as plain text.

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
curl -L -o /tmp/user.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/user-v4.zip
cd /app
unzip /tmp/user.zip
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
vim /etc/systemd/system/user.service
```

```ini
[Unit]
Description=User Service
After=network-online.target

[Service]
User=roboshop
Environment=REDIS_URL="redis://<REDIS-SERVER-IPADDRESS>:6379"
Environment=MONGO_URL="mongodb://<MONGODB-SERVER-IPADDRESS>:27017/users"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=user
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<REDIS-SERVER-IPADDRESS>` and `<MONGODB-SERVER-IPADDRESS>` with the private IPs of those servers.**

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable user
systemctl start user
```

---

## Verification

```shell
systemctl status user
journalctl -u user -f
curl http://localhost:8080/health
```

`/health` returns `{"app":"OK","mongo":true,"redis":true}`. If either database is down you get `503`, and the log shows which one.

Try a login with the demo user (loaded by catalogue's master data):

```shell
curl -i -X POST http://localhost:8080/login -H 'Content-Type: application/json' -d '{"name":"roboshop","password":"RoboShop@1"}'
```

`200` means it works. A wrong password gives `401`. The password is never written to the log.

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | 200, or 503 if MongoDB or Redis is down |
| GET | `/uniqueid` | a new guest id from Redis |
| POST | `/register` | `{name, email, password}` → 201, 400 missing fields, 409 name taken |
| POST | `/login` | `{name, password}` → 200, 401 wrong login |
| GET | `/check/<name>` | 200 or 404, used by payment |
| POST | `/order/<name>` | saves an order, used by payment |
| GET | `/history/<name>` | order history |
| GET | `/users` | all users, without passwords |
| GET | `/metrics` | Prometheus metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend` and `roboshop-payment`.
