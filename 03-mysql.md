# MySQL

The developer has chosen MySQL for the shipping data: all countries, their cities and the location of each city. Shipping uses the location to work out the distance and the shipping cost.

> **Check with the developer for the exact version required. The developer has shared the version as MySQL 8.4.**

---

## Install

By default MySQL 8.0 is installed. Enable 8.4 and install it:

**You can list modules by using `dnf module list mysql`**

```shell
dnf module enable mysql:8.4 -y
dnf install mysql-server -y
```

Start & enable the MySQL service:

```shell
systemctl enable mysqld
systemctl start mysqld
```

---

## Configure

A new MySQL has no root password: `mysql -uroot` logs in without one. Set the root password. Use **`RoboShop@1`** or another password of your choice:

```shell
mysql -uroot -e "
ALTER USER 'root'@'localhost' IDENTIFIED BY 'RoboShop@1';
CREATE USER IF NOT EXISTS 'root'@'%' IDENTIFIED BY 'RoboShop@1';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
"
```

- `'root'@'localhost'`: root login on this server.
- `'root'@'%'`: root login from other servers. Shipping loads the data from its own server, so it needs this.

After this, `mysql -uroot` without a password fails with `Access denied`. That is expected.

> **Type `-pRoboShop@1` without a space.** With a space, `-p RoboShop@1` asks for the password and treats `RoboShop@1` as a database name.

> **MySQL 8.4 change:** the old `mysql_native_password` login method is turned off by default. The app user is created with the newer default (`caching_sha2_password`), which the shipping service supports. No extra setting is needed.

---

## Verification

```shell
systemctl status mysqld
ss -lntp | grep 3306
mysql -u root -pRoboShop@1 -e "SELECT VERSION();"
```

The data and the `shipping` app user are loaded later from the shipping server, see [08-shipping.md](08-shipping.md). After that, check the data:

```shell
mysql
```

```sql
SHOW DATABASES;
```

---

## Security Group

Allow **inbound TCP 3306** from `roboshop-shipping` only.
