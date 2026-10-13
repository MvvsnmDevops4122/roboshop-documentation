# Shipping

Shipping works out how far a package has to travel and what it costs to ship it. It looks up the city in MySQL, measures the distance from the warehouse, and adds the shipping cost to the cart.

Shipping is written in Java with Spring Boot 4. **Maven** is the Java build tool: it downloads the libraries the application needs and packages everything into one `.jar` file.

> **Developer has chosen Java and Maven. Check with the developer which versions are needed. This setup requires Java >= 25 and Maven >= 3.9.**

---

## Install Java and Maven

**You can list modules by using `dnf module list maven`**

```shell
dnf module enable maven:3.9 -y
dnf install maven java-25-openjdk-devel -y
```

`dnf install maven` also brings an older Java (21), and Maven uses it by default. This application needs Java 25, so make Java 25 the default **once**:

```shell
alternatives --config java
```

It lists the installed Java versions. Type the number of the line with `java-25-openjdk` and press Enter. Now the `java` command points to Java 25.

Maven needs `JAVA_HOME`, the Java 25 folder. The folder name contains the full version, so take it from the `java` command instead of typing it:

```shell
echo "export JAVA_HOME=$(dirname $(dirname $(readlink -f /usr/bin/java)))" > /etc/profile.d/java.sh
source /etc/profile.d/java.sh
echo $JAVA_HOME
```

- `readlink -f /usr/bin/java` gives the real file, for example `/usr/lib/jvm/java-25-openjdk-25.0.1.0.8-1.el9.x86_64/bin/java`. The two `dirname` remove `/bin/java` from the end.
- Files in `/etc/profile.d/` run at every login, so you never have to type it again, even after a reboot.

Verify:

```shell
java -version
mvn -v
```

Both should show version `25`.

---

## Create Application User

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

---

## Download the Application

```shell
mkdir /app
curl -L -o /tmp/shipping.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/shipping-v4.zip
cd /app
unzip /tmp/shipping.zip
```

---

## Build the Application

```shell
cd /app
mvn clean package
mv target/shipping.jar shipping.jar
```

The first build takes a few minutes, because Maven downloads all the libraries.

---

## Load Master Data

Load the data **before** starting the service. This includes all countries and their cities with their locations. The developer provides the schema and the app user in the `db` folder.

Install the MySQL client:

```shell
dnf install mysql -y
```

Load the cities. This creates the `cities` database and its tables:

```shell
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/master-data.sql
```

Create the app user. The application logs in as `shipping`, not as `root`, and it can only use the `cities` database:

```shell
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/app-user.sql
```

The import is large (about 60 MB) and takes a minute. It is safe to run again: it drops and recreates the tables.

---

## Configure SystemD Service

```shell
vim /etc/systemd/system/shipping.service
```

```ini
[Unit]
Description=Shipping Service
After=network-online.target

[Service]
User=roboshop
Environment=CART_ENDPOINT=<CART-SERVER-IPADDRESS>:8080
Environment=DB_HOST=<MYSQL-SERVER-IPADDRESS>
Environment=DB_USER=shipping
Environment=DB_PASSWORD=RoboShop@1
ExecStart=/bin/java -jar /app/shipping.jar
SyslogIdentifier=shipping
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<CART-SERVER-IPADDRESS>` and `<MYSQL-SERVER-IPADDRESS>` with the private IPs of those servers.**

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable shipping
systemctl start shipping
```

---

## Verification

Spring Boot takes 10-20 seconds to start. Watch the log until you see `Started ShippingApplication`:

```shell
systemctl status shipping
journalctl -u shipping -f
```

```shell
curl http://localhost:8080/actuator/health
curl http://localhost:8080/count
curl http://localhost:8080/codes
```

`/actuator/health` returns `{"status":"UP"}` and includes the database status. `/count` returns the number of cities.

| Problem | Meaning |
|---------|---------|
| Log shows `Access denied for user 'shipping'` | `app-user.sql` was not loaded |
| Log shows `Communications link failure` | Shipping cannot reach MySQL: check the IP and port 3306 in the security group |
| Log shows `Cannot find any provider supporting RSA/ECB/OAEPWithSHA-1AndMGF1Padding` | Old `shipping-v4.zip` that connects to MySQL without TLS. Download the zip again and rebuild |
| `503` from `/codes` | MySQL went down while shipping was running |
| `502` from `/confirm/...` | Shipping cannot reach Cart |

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | `OK` |
| GET | `/codes` | list of countries |
| GET | `/match/<code>/<text>` | city search, needs at least 3 letters, otherwise 400 |
| GET | `/calc/<cityId>` | `{distance, cost}`, 404 for an unknown city |
| POST | `/confirm/<id>` | adds shipping to the cart |
| GET | `/count` | number of cities |
| GET | `/memory` / `/free` | memory leak demo: each call to `/memory` keeps 25 MB more |
| GET | `/actuator/health`, `/actuator/prometheus` | Spring Boot health and metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend` only.
