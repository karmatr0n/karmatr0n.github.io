---
layout: post
title: "MongoDB Replica Set Configuration with Authentication and Transactions Support"
date:  2023-02-28 10:30:00
tags:  mongodb transactions replica_set authentication 
---

I've been working on a Python project that requires MongoDB with transactions in recent months.
The transactions feature is only available for replica sets and sharded clusters. So, the goal of 
this post is to provide a quick guide for configuring a development environment 
with authentication and transaction enabled, rather than a fully functional cluster configuration.

## Prerequisites

* [CentOS 9](https://centos.org/stream9/)

## Updating your system
{% highlight bash linenos %}
$ sudo dnf update
{% endhighlight %}

## Installing MongoDB
{% highlight bash linenos %}
$ sudo vi /etc/yum.repos.d/mongodb.repo
[mongodb-org-6.0] 
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9Server/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc


$ sudo dnf install mongodb-org mongodb-mongosh
$ sudo systemctl start mongod
{% endhighlight %}

## Creating the DB and adding user with the required roles
{% highlight bash linenos %}
$ mongosh
> use <db_name>
> db.createUser({ user: <db_user>, pwd: <db_password>, roles: [{ role: 'dbOwner', db: <db_name> }] })
{% endhighlight %}

## Adding the keyfile required for the Replica Set
{% highlight bash linenos %}
$ sudo openssl rand -base64 768 >  /tmp/keyFile
$ sudo mv /tmp/keyFile /var/lib/mongo/
$ sudo chown mongod:mongod /var/lib/mongo/keyFile 
$ sudo chmod 600 /var/lib/mongo/keyFile
{% endhighlight %}

## Configuring the replica set and authentication for MongoDB
{% highlight bash linenos %}
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
{% endhighlight %}

## Testing the database connection
{% highlight bash linenos %}
$ mongosh 'mongodb://<db_user>:<db_password>@127.0.0.1:27017/<db_name>'
{% endhighlight %}

## Initializing the replica set
Once connected into the database you can initialize the replica set and check the status.
{% highlight js linenos %}
rs.initiate()
rs.status()
{% endhighlight %}

## Conclusion
After following the steps above you should have a MongoDB replica set with authentication and transactions
enabled. Example:
{% highlight js linenos %}
session = db.getMongo().startSession( { readPreference: { mode: "primary" } } );

employeesCollection = session.getDatabase("hr").employees;
eventsCollection = session.getDatabase("reporting").events;
session.startTransaction( { readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } } );

try {
  employeesCollection.updateOne( { employee: 3 }, { $set: { status: "Inactive" } } );
  eventsCollection.insertOne( { employee: 3, status: { new: "Inactive", old: "Active" } } );
} catch (error) {
  session.abortTransaction();
  throw error;
}

session.commitTransaction();
session.endSession();
{% endhighlight %}

## References

* [MongoDB - Deploy a Replica Set](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/)
* [MongoDB - Enable Access Control](https://www.mongodb.com/docs/manual/tutorial/enable-authentication/)
* [MongoDB - db.createUser](https://www.mongodb.com/docs/manual/reference/method/db.createUser/)
* [MongoDB - Database Administration Roles](https://www.mongodb.com/docs/manual/reference/built-in-roles/#database-administration-roles)
* [MongoDB - Transactions](https://www.mongodb.com/transactions)