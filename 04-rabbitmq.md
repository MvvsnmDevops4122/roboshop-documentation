# RabbitMQ

RabbitMQ is a message queue. Payment puts each new order in the `orders` queue, and Dispatch takes orders out of the queue and ships them. Payment does not wait for Dispatch: the order is accepted as soon as it is in the queue, even if Dispatch is down. Orders wait safely in the queue until Dispatch is back.

> **Check with the developer for the exact version required. The developer has shared the version as RabbitMQ 4.x.**

---

## Install

RabbitMQ is not in the RHEL repos. Set up the RabbitMQ repo file, which also provides the Erlang version RabbitMQ needs:

```shell
vim /etc/yum.repos.d/rabbitmq.repo
```

```ini
[modern-erlang]
name=modern-erlang-el9
baseurl=https://yum1.rabbitmq.com/erlang/el/9/$basearch
        https://yum2.rabbitmq.com/erlang/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-noarch]
name=modern-erlang-el9-noarch
baseurl=https://yum1.rabbitmq.com/erlang/el/9/noarch
        https://yum2.rabbitmq.com/erlang/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/$basearch
        https://yum1.rabbitmq.com/rabbitmq/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9-noarch]
name=rabbitmq-el9-noarch
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/noarch
        https://yum1.rabbitmq.com/rabbitmq/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md
```

`gpgcheck=1` makes `dnf` verify that the packages are really signed by the RabbitMQ team. `dnf` downloads and imports the keys listed in `gpgkey` the first time.

Install Erlang and RabbitMQ:

```shell
dnf install erlang rabbitmq-server -y
```

Start & enable the RabbitMQ service:

```shell
systemctl enable rabbitmq-server
systemctl start rabbitmq-server
```

---

## Configure

RabbitMQ comes with a default user `guest/guest`, but `guest` can only connect from the same server. Create a user for the application:

```shell
rabbitmqctl add_user roboshop roboshop123
rabbitmqctl set_permissions -p / roboshop ".*" ".*" ".*"
```

Optional: turn on the management web UI (port 15672) to watch the queue:

```shell
rabbitmq-plugins enable rabbitmq_management
rabbitmqctl set_user_tags roboshop administrator
```

Open `http://<RABBITMQ-PUBLIC-IP>:15672` and log in as `roboshop / roboshop123`. Add port 15672 from your IP to the security group first.

---

## Verification

```shell
systemctl status rabbitmq-server
ss -lntp | grep 5672
rabbitmqctl list_users
```

After the first order, the exchange and queue exist:

```shell
rabbitmqctl list_exchanges | grep robot-shop
rabbitmqctl list_queues name messages consumers
```

`orders` should show `0` messages and `1` consumer while Dispatch is running. Stop Dispatch and place an order: the message count goes up. Start Dispatch again and it goes back to `0`.

---

## Security Group

Allow **inbound TCP 5672** from `roboshop-payment` and `roboshop-dispatch` only.
