# Catalogue

Catalogue is a microservice that serves the list of products shown in the RoboShop website, with search, categories and product ratings.

> **Developer has chosen Node.js. Check with the developer which version is needed. This setup requires Node.js >= 24.**

---

## Install Node.js
C:\devops\daws-92s\repos\roboshop-documentation
By default Node.js 16 is enabled. Enable version 24 and install it:

**You can list modules by using `dnf module list nodejs`**

```shell
dnf module disable nodejs -y
dnf module enable nodejs:24 -y
dnf install nodejs -y
```

Verify:

```shell
node -v
```

---

## Create Application User

Applications should run as a non-root user:

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

User **roboshop** is a system (daemon) user that only runs the application. Nobody logs in with it: the `/sbin/nologin` shell blocks logins and it has no password.

---

## Download the Application

We keep the application in one standard location, `/app`:

```shell
mkdir /app
curl -L -o /tmp/catalogue.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/catalogue-v4.zip
cd /app
unzip /tmp/catalogue.zip
```

---

## Install Dependencies

Every application uses common libraries written by others. The developer lists them in `package.json`. Let's download them:

```shell
cd /app
npm install
```

`npm install` downloads the libraries into the `node_modules` folder.

---

## Configure SystemD Service

```shell
vim /etc/systemd/system/catalogue.service
```

```ini
[Unit]
Description=Catalogue Service
After=network-online.target

[Service]
User=roboshop
Environment=MONGO_URL="mongodb://<MONGODB-SERVER-IPADDRESS>:27017/catalogue"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<MONGODB-SERVER-IPADDRESS>` with the private IP of the MongoDB server.**

Optional settings:

| Variable | Default | Meaning |
|----------|---------|---------|
| `GO_SLOW` | `0` | Delay in ms for `GET /product/<sku>`, to practise slow-response troubleshooting |
| `LOG_LEVEL` | `info` | `debug` shows more log lines |

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable catalogue
systemctl start catalogue
```

`daemon-reload` makes systemd read the new service file.

---

## Load Master Data

The application needs the list of products before it can sell anything. The developer provides the schema and indexes in `db/master-data.js`. Master data, like the product list, usually comes from the business team.

The same file also creates a demo user (`roboshop` / `RoboShop@1`) in the `users` database.

Install the MongoDB client. Create the same repo file as on the MongoDB server:

```shell
vim /etc/yum.repos.d/mongo.repo
```

```ini
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
enabled=1
gpgcheck=0
```

```shell
dnf install mongodb-mongosh -y
```

Load the data **only once**. Loading it again adds the products twice:

```shell
mongosh --host <MONGODB-SERVER-IPADDRESS> </app/db/master-data.js
```

To check whether it is already loaded:

```shell
mongosh --host <MONGODB-SERVER-IPADDRESS> --quiet --eval 'db.getSiblingDB("catalogue").products.countDocuments()'
```

`11` means the data is there.

---

## Verification

```shell
systemctl status catalogue
journalctl -u catalogue -f
curl http://localhost:8080/health
curl http://localhost:8080/products
```

`/health` returns `{"app":"OK","mongo":true}`. If MongoDB is not reachable you get `503` with `"mongo":false`.

Look inside the database:

```shell
mongosh --host <MONGODB-SERVER-IPADDRESS>
```

```js
show dbs
use catalogue
show collections
db.products.find()
```

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | 200, or 503 if MongoDB is down |
| GET | `/products` | all products |
| GET | `/product/<sku>` | one product, 404 if unknown |
| GET | `/products/<category>` | products in a category |
| GET | `/categories` | list of categories |
| GET | `/search/<text>` | full-text search |
| GET | `/ratings` | ratings of all products |
| GET | `/ratings/<sku>` | rating of one product |
| PUT | `/rate/<sku>/<score>` | rate a product, score 1-5 |
| GET | `/metrics` | Prometheus metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend` and `roboshop-cart`.

> **NOTE:** Note down the private IP of this server. The frontend Nginx needs it, see [11-frontend.md](11-frontend.md).
