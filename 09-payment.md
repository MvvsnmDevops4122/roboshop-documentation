# Payment

Payment handles the checkout. For each order it:

1. checks the user with the User service
2. calls the payment gateway
3. puts the order in the RabbitMQ `orders` queue, where Dispatch picks it up
4. saves the order in the user's history and empties the cart

Payment is written in Python with Flask. Flask's built-in server is only for development, so in production **gunicorn** runs the Flask application. It:

- listens on port 8080 and passes each request to the Flask code
- runs workers and threads, so many requests are handled at the same time
- restarts a worker when it crashes

It does the same job as PM2 for Node.js. It replaces uWSGI from the older version, and it is pure Python, so no compiler (`gcc`) is needed anymore.

> **Developer has chosen Python. Check with the developer which version is needed. This setup requires Python >= 3.14.**

---

## Install Python

RHEL 9 comes with Python 3.9 as `python3`. Newer versions are installed next to it with their own name, so system tools that use 3.9 keep working:

```shell
dnf install python3.14 python3.14-pip -y
python3.14 --version
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
curl -L -o /tmp/payment.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/payment-v4.zip
cd /app
unzip /tmp/payment.zip
```

---

## Install Dependencies

Install the libraries in a **virtual environment** (venv). A venv is a folder with its own Python and its own libraries. The application gets the exact library versions in `requirements.txt`, and nothing is mixed with the Python packages of the OS.

```shell
cd /app
python3.14 -m venv /app/.venv
/app/.venv/bin/pip install -r requirements.txt
```

---

## Configure SystemD Service

```shell
vim /etc/systemd/system/payment.service
```

```ini
[Unit]
Description=Payment Service
After=network-online.target

[Service]
User=roboshop
WorkingDirectory=/app
Environment=CART_HOST=<CART-SERVER-IPADDRESS>
Environment=CART_PORT=8080
Environment=USER_HOST=<USER-SERVER-IPADDRESS>
Environment=USER_PORT=8080
Environment=AMQP_HOST=<RABBITMQ-SERVER-IPADDRESS>
Environment=AMQP_USER=roboshop
Environment=AMQP_PASS=roboshop123
ExecStart=/app/.venv/bin/gunicorn --config gunicorn.conf.py --no-control-socket payment:app
SyslogIdentifier=payment
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<CART-SERVER-IPADDRESS>`, `<USER-SERVER-IPADDRESS>` and `<RABBITMQ-SERVER-IPADDRESS>` with the private IPs of those servers.**

The service now runs as `roboshop` instead of `root`. `payment:app` means "the `app` object in `payment.py`".

`--no-control-socket`: gunicorn 25 and newer opens a control socket in the home folder of the user, here `/app/.gunicorn`. `roboshop` cannot write to `/app`, so the log shows `Control server error: [Errno 13] Permission denied`. The application does not need this socket, so it is turned off.

Optional settings:

| Variable | Default | Meaning |
|----------|---------|---------|
| `PAYMENT_GATEWAY` | `https://www.google.com/` | URL called as a fake payment gateway. Leave it empty to skip the call |
| `PAYMENT_DELAY_MS` | `0` | Extra delay per payment, to practise slow-response troubleshooting |

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable payment
systemctl start payment
```

---

## Verification

```shell
systemctl status payment
journalctl -u payment -f
curl http://localhost:8080/health
```

`/health` returns `{"app":"OK","rabbitmq":true}`. If RabbitMQ is not reachable you get `503` with `"rabbitmq":false`.

Place an order on the website. The checkout returns `201 Created` with an order id.

| Status from `/pay/<id>` | Meaning |
|-------------------------|---------|
| `201` | Order accepted and queued |
| `400` | The cart is empty, or shipping was not added |
| `502` | Payment cannot reach User or the payment gateway |
| `504` | User or the gateway did not answer in time |
| `503` | RabbitMQ is down, so the order cannot be queued |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend` only.
