---
layout: post
title: "Configuring MongoDB for a Replica Set with Authentication and transactions support"
date:  2023-02-28 10:30:00
tags:  mongodb transactions replica_set authentication 
---

# Introduction

In the recent months I have been developing a python application that requires MongoDB 
with transactions. The transactions feature is just available for replica sets and sharded clusters, 
and this is just a simple guide to setup a development environment with authentication and 
transactions support but not for a real cluster.

# Prerequisites

* [CentOS 9](https://centos.org/stream9/)

# Update your system
```
$ sudo dnf update
```

# Install MongoDB
```
$ sudo vi /etc/yum.repos.d/mongodb.repo
[mongodb-org-6.0] 
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9Server/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc


$ sudo dnf install mongodb-org mongodb-mongosh
$ sudo systemctl start mongod
```

# Create DB and add user with the required roles
```
$ mongosh
> use <db_name>
> db.createUser({ user: <db_user>, pwd: <db_password>, roles: [{ role: 'dbOwner', db: <db_name> }] })
```

# Add the keyfile required for the Replica Set
```
$ sudo openssl rand -base64 768 >  /tmp/keyFile
$ sudo mv /tmp/keyFile /var/lib/mongo/
$ sudo chown mongod:mongod /var/lib/mongo/keyFile 
$ sudo chmod 600 /var/lib/mongo/keyFile

```

# Configure replica set and authentication for MongoDB
```
$ sudo systemctl stop mongod
$ sudo vi /etc/mongod.conf

security:
  authorization: enabled
  keyFile: /var/lib/mongo/keyFile

replication:
  replSetName: "rs0"

$ sudo systemctl disable mongod
$ sudo systemctl enable mongod
$ sudo systemctl start mongod
```

# How to test the database
```
$ mongosh 'mongodb://<db_user>:<db_password>@127.0.0.1:27017/<db_name>'
```

# References

* [MongoDB - Deploy a Replica Set](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/)
* [MongoDB - Enable Access Control](https://www.mongodb.com/docs/manual/tutorial/enable-authentication/)
* [MongoDB - db.createUser](https://www.mongodb.com/docs/manual/reference/method/db.createUser/)
* [MongoDB - Database Administration Roles](https://www.mongodb.com/docs/manual/reference/built-in-roles/#database-administration-roles)