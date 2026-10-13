# Redis

Redis (REmote DIctionary Server) is an open-source, in-memory data store used as a database and cache. It is very fast because it keeps data in RAM. It uses a simple key-value model but supports complex data types.

RoboShop uses Redis for:

| Used by | What it stores |
|---------|----------------|
| Cart | Every cart, as one key per user (expires after 1 hour without changes) |
| User | A counter that gives each guest visitor a unique id (`anonymous-1`, `anonymous-2`, ...) |

> **Check with the developer for the exact version required. This setup uses Redis 7.**
>
> Redis 7 is the newest Redis on RHEL 9. Newer RHEL releases also offer **Valkey 8**, an open-source fork of Redis that works with the same clients.

---

## Install

By default Redis 6 is enabled. Enable version 7 and install it:

**You can list modules by using `dnf module list redis`**

```shell
dnf module disable redis -y
dnf module enable redis:7 -y
dnf install redis -y
```

---

## Configure

By default Redis listens only on `localhost (127.0.0.1)`. User and Cart run on other servers, so Redis must listen on all addresses.

Edit the config file:

```shell
vim /etc/redis/redis.conf
```

- Update `bind 127.0.0.1 -::1` to `bind 0.0.0.0 -::1`
- Update `protected-mode yes` to `protected-mode no`

Or do the same with one command:

```shell
sed -i -e 's/^bind 127.0.0.1/bind 0.0.0.0/' -e 's/^protected-mode yes/protected-mode no/' /etc/redis/redis.conf
```

Start & enable the Redis service:

```shell
systemctl enable redis
systemctl start redis
```

---

## Verification

```shell
systemctl status redis
ss -lntp | grep 6379
redis-cli ping
```

`redis-cli ping` should answer `PONG`.

After you add something to the cart on the website, you can see it in Redis:

```shell
redis-cli keys '*'
redis-cli get <user-id>
```

---

## Security Group

Allow **inbound TCP 6379** from `roboshop-user` and `roboshop-cart` only.
