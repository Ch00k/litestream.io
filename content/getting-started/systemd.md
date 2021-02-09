---
title : "Running as a systemd service"
date: 2021-02-01T00:00:00Z
layout: docs
menu:
  docs:
    parent: "getting-started"
weight: 220
---

This tutorial will get you running Litestream as a systemd service on Debian-based
operating systems such as Ubuntu. Running as a background service means that
Litestream will always run in the background and restart automatically if the
server restarts. It assumes you are comfortable with using a command line.

_This tutorial should take approximately 10 minutes to complete._


## Prerequisites

This tutorial assumes you have read the [Replicating a single database](/getting-started/basic/)
tutorial already. Please read that to understand the basic operation of Litestream.


### Install Litestream & SQLite

Before continuing, [please install Litestream on your local machine](/install).

You will also need [SQLite](https://sqlite.org/) installed for this tutorial. It
comes packaged with some operating systems such as Mac OS X but you may need to
install it separately.


### Creating an S3 bucket

If you don't already have an Amazon AWS account, you can go 
[https://aws.amazon.com/](https://aws.amazon.com/) and click "Create Account".
Once you have an account, you'll log in and obtain API credentials from their
IAM service. You should have an "access key id" and a "secret access key".
Once you have those, add them to your environment variables. On Mac OS X &
Linux, you can run this from your command line using your own key values:

```sh
export AWS_ACCESS_KEY_ID=AKIAxxxxxxxxxxxxxxxx
export AWS_SECRET_ACCESS_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/xxxxxxxxx
```

After your credentials are set, you'll also need to create a bucket in the S3
service of AWS. You'll need to create a unique name for your bucket. In this
tutorial, we'll name our bucket, `"mybkt.litestream.io"`.


## Configuration file

When running as a systemd service, we'll configure Litestream using a
[configuration file](/reference/config) instead of command line flags so we
don't need to edit the service definition. The default path for the Litestream
configuration is `/etc/litestream.yml`

Litestream monitors one or more _databases_ and each of those databases replicates
to one or more _replicas_. First, we'll create a basic configuration file. Make
sure to replace replace your AWS credentials with your own, the bucket name with
your bucket name, and update `MYUSER` to your local Linux username.

```
sudo cat > /etc/litestream.yml <<EOF
access-key-id:     AKIAxxxxxxxxxxxxxxxx
secret-access-key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx/xxxxxxxxx

dbs:
  - path: /home/MYUSER/friends.db
    replicas:
      - url: s3://mybkt.litestream.io/friends.db
      - sync-interval: 1s
EOF

```

This configuration specifies that we want want Litestream to monitor our
`friends.db` database in our home directory and continuously replicate it to
our `mybkt.litestream.io` S3 bucket.

After changing our configuration, we'll need to restart the Litestream service
for it to register the change:

```
sudo systemctl restart litestream
```

## Writing to our database

Now that Litestream is running in the background, let's create our `friends.db`
database:

```sh
sqlite3 friends.db
```

We'll create a simple table to store our friends' names:

```sql
CREATE TABLE friends (name TEXT);
```

Then we'll insert some rows:

```
INSERT INTO friends (name) VALUES ('Cory');
INSERT INTO friends (name) VALUES ('Kelly');
```


## Simulating a disaster

Next, we'll simulate a catastrophic server failure by stopping Litestream:

```sh
sudo systemctl stop litestream
```

And then we'll delete our database:

```sh
rm friends.db friends.db-shm friends.db-wal
```

This is the state our server would be in if it had crashed and we had rebuilt it
but lost our data.


## Restoring our database

Our Litestream service and configuration file are still in place so we can
easily restore our database. Simply run the `restore` command with our database
path:

```sh
litestream restore friends.db
```

Litesteam will find the database in the configuration file and restore the most
recent copy it has from its S3 replica.

You can verify that your data is all there by connecting with SQLite:

```
sqlite3 friends.db
```

And querying for your data:

```
SELECT * FROM friends
Cory
Kelly
```


## Further reading

You now have a production-ready replication setup using SQLite and Litestream.
Please see the [How-To Guides](/guides) and [Reference](/reference) sections
for more details about how to configure and run Litestream.

