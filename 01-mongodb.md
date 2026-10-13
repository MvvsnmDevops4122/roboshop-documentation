# MongoDB

MongoDB is a popular NoSQL database designed for handling large volumes of unstructured or semi-structured data. Instead of using tables like in traditional SQL databases, MongoDB stores data in flexible, JSON-like documents.

## 🗃️ SQL vs NoSQL (MongoDB) Comparison

| Feature | SQL (Relational DB) | NoSQL (MongoDB - Document DB) |
|---------|---------------------|-------------------------------|
| **Data Structure** | Tables with rows and columns | Collections with JSON-like documents (BSON) |
| **Schema** | Rigid schema (must define columns) | Dynamic schema (schema-less, flexible) |
| **Query Language** | SQL (Structured Query Language) | MQL (MongoDB Query Language - JavaScript style) |
| **Best Use Cases** | Banking, ERP, CRM, complex relational systems | Real-time analytics, IoT, content management, apps |
| **Examples** | MySQL, PostgreSQL, Oracle, SQL Server | MongoDB, CouchDB, Amazon DocumentDB |

**Example Document:**

```json
{
  "_id": "6638fa12345abc",
  "name": "Alice",
  "age": 30,
  "skills": ["Java", "DevOps"],
  "address": {
    "city": "Hyderabad",
    "zip": "500001"
  }
}
```

RoboShop uses two MongoDB databases:

| Database | Used by | Collections |
|----------|---------|-------------|
| `catalogue` | Catalogue | `products`, `ratings` |
| `users` | User | `users`, `orders` |

> **Check with the developer for the exact version required. The developer has shared the version as MongoDB 7.0.**
>
> MongoDB 8.0 crashes on the RHEL 9 practice AMI (`Fatal assertion 40379` at startup), so stay on 7.0.

---

## Install

Set up the MongoDB repo file:

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

Install MongoDB:

```shell
dnf install mongodb-org -y
```

Start & enable the MongoDB service:

```shell
systemctl enable mongod
systemctl start mongod
```

---

## Configure

By default MongoDB listens only on `localhost (127.0.0.1)`, so only applications on this same server can reach it. Catalogue and User run on other servers, so MongoDB must listen on all addresses.

Update the listen address from `127.0.0.1` to `0.0.0.0` in `/etc/mongod.conf`:

```shell
vim /etc/mongod.conf
```

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0
```

Or do the same with one command:

```shell
sed -i 's/127.0.0.1/0.0.0.0/' /etc/mongod.conf
```

Restart the service to apply the change:

```shell
systemctl restart mongod
```

---

## Verification

```shell
systemctl status mongod
ss -lntp | grep 27017
```

The port should show `0.0.0.0:27017`. The data (products and a demo user) is loaded later from the catalogue server, see [05-catalogue.md](05-catalogue.md).

---

## Security Group

Allow **inbound TCP 27017** from `roboshop-catalogue` and `roboshop-user` only.
