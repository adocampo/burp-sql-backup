# BURP database backups

[BURP](https://burp.grke.org) is an open source file-level backup tool. It has plenty of options, like compression, deduplication, and much much more other options.

It can even take advantage on LVM snapshots in Linux or VSS on Windows, so the backup will run smoothly accessing a read-only copy of the file system, avoiding inconsistences, but it is not designed to backup databases.

Fortunately, burp has mechanisms for pre-backup and post-backup, called `backup_script_pre` and `backup_script_post` respectively. So if there is a need to backup a SQL database, on the burp client settings we can call those scripts which will be the responsible for making the database dump _before_ the backup process starts. When the backup process ends, we can run the `backup_script_post` to, for example, delete the dumps generated previously.

Currently, this repository has the scripts to make the SQL dump for the databases

- MySQL/MariaDB
- PostgreSQL

## Prerequisites

We'll need to have installed burp both in the server and clients.
We also need to impersonate as the `posgres` user, so `sudo` needs to be installed.
We will be using the command `run-parts` present in all modern distros.

## Installation

To install burp, we can use the package on the Linux repository (often outdated) or compile it through its [sources](https://github.com/grke/burp/tags). On Windows, we need to use the [precomplied Windows binaries](https://github.com/grke/burp/tags).

### Installation from sources

Installing burp from sources is pretty straight forward, first we need to download burp tarball with

```
wget https://github.com/grke/burp/archive/refs/tags/2.5.4.tar.gz
```

Then untar it and enter the source code tree

```
tar zxvf 2.5.4.tar.gz
cd burp-2.5.4
```

At this point, we can read the README file located in there, which usually can be resumed in the following steps in debian-based distros.

```
apt-get install make pkg-config check g++ \
                librsync-dev libz-dev libssl-dev uthash-dev \
                autoconf automake libtool
autoreconf -vif
./configure --prefix=/usr --sysconfdir=/etc/burp --localstatedir=/var
make
make install
make install-configs
```

NOTE: It's out of the scope of this document explain how to configure both server and client and make backups or restorations. Please, refer the [official documentation](https://burp.grke.org/docs.html) for that.

## Set up burp client

When our client is a database server, we'll need to set up a proper configuration. That is place the database dump scripts and tell the burp client configuration file (`burp.conf`) to execute them.

In this repo are present the database dump scripts and an example of `burp.conf`. You'll need to copy and modify the `etc/burp/burp.conf` file and place it where your `burp.conf` lives, usually at `/etc/burp/burp.conf`. On the other hand, you will need to place the database backup scripts somewhere accesible and readable. I selected the path `/usr/share/burp/scripts/backup_post` and `/usr/share/burp/scripts/backup_pre`. If you want another place, make sure to change accordingly on your `burp.conf`.

That's it. We will take advantage of the linux tool [run-parts](https://manpages.debian.org/bullseye/debianutils/run-parts.8.en.html), which essentially will run any script placed in a given directory.
NOTE: scripts must not have any extension in order to work, so a script named `foo.sh` must be renamed to `foo`.

When a backup starts, before even indexing files, burp will try to execute the `backup_script`, which is just `run-parts` and as we are on the pre_backup step, will take as arguments for that command the path where database dumps are.

Then will run them and if there is no error, then will continue the backup process. If any error occurs, the backup will fail.

Finally, when the backup ends, it will run again the `backup_script`, taking as argument this time the path were the delete backups older than 7 days is.

## Credentials file

For security reasons, credentials are not hardcoded into the scripts, but we use mechanisms MySQL has through the file `my.cnf`. If you already have one, take a look into `root/.my.cnf` to see how to specify your own credentials for `mysql` and `mysqldump`. Make sure credentials file has the proper permissions.

```
chmod 600 ~/.my.cnf
```

You can see if the credentials are properly working with the commands

```
mysql --defaults-file=/root/.my.cnf --print-defaults
mysqldump --defaults-file=/root/.my.cnf --print-defaults
```

PosgreSQL doesn't need credentials at all if the SQL server is localhost, as we use

## Roadmap

- Notify via email/webhook if a backup has failed because the database dump.
- Add support for MSSQL
