# Dispatch

Dispatch ships the products after a purchase. It reads paid orders from the RabbitMQ `orders` queue, packs them, and logs a tracking id for each order.

Dispatch has no web port. It only connects out to RabbitMQ and waits for messages. If RabbitMQ is down, it retries every 5 seconds.

It is written in Go. Go builds the code into **one binary file** that runs without Go or any libraries installed.

> **Developer has chosen Go. Check with the developer which version is needed. This setup requires Go >= 1.25.**

---

## Install Go

```shell
dnf install golang -y
go version
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
curl -L -o /tmp/dispatch.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/dispatch-v4.zip
cd /app
unzip /tmp/dispatch.zip
```

---

## Build the Application

The developer ships `go.mod` and `go.sum`, so there is no need for `go mod init` or `go get`:

| Go | Node.js | What it holds |
|----|---------|---------------|
| `go.mod` | `package.json` | Module name, Go version, the libraries and their versions |
| `go.sum` | `package-lock.json` (partly) | A checksum of every library |

In Node.js, `package.json` usually has version ranges like `^4.18.0`, and `package-lock.json` fixes the exact version. In Go, `go.mod` already has exact versions like `v1.10.0`. `go.sum` only checks that a downloaded library was not changed: if the checksum does not match, the build stops. Both files are committed to Git.

```shell
cd /app
go build -o dispatch .
```

`go build` downloads the libraries, checks them against `go.sum`, and creates the binary `/app/dispatch`.

---

## Configure SystemD Service

```shell
vim /etc/systemd/system/dispatch.service
```

```ini
[Unit]
Description=Dispatch Service
After=network-online.target

[Service]
User=roboshop
Environment=AMQP_HOST=<RABBITMQ-SERVER-IPADDRESS>
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123
ExecStart=/app/dispatch
SyslogIdentifier=dispatch
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<RABBITMQ-SERVER-IPADDRESS>` with the private IP of the RabbitMQ server.**

Optional: `DISPATCH_DELAY_MS` (default `500`) is the time to pack one order. Set it high, for example `60000`, to see orders pile up in the queue.

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable dispatch
systemctl start dispatch
```

---

## Verification

```shell
systemctl status dispatch
journalctl -u dispatch -f
```

At start you see:

```json
{"level":"INFO","msg":"connected to RabbitMQ, waiting for orders","service":"dispatch","queue":"orders"}
```

Place an order on the website. Two lines appear, with the same `requestId` as the payment logs:

```json
{"level":"INFO","msg":"order received","service":"dispatch","requestId":"...","orderid":"...","user":"roboshop","items":2,"total":2084.2}
{"level":"INFO","msg":"order dispatched","service":"dispatch","requestId":"...","orderid":"...","trackingId":"RS0123456789","durationMs":501}
```

If you see `lost connection to RabbitMQ, retrying in 5s`, check the IP, the user and password, and port 5672 in the RabbitMQ security group.

---

## Security Group

No inbound rules except SSH. Dispatch only makes outbound connections.
