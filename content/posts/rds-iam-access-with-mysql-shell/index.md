---
title: "RDS/Aurora IAM DB access with MySQL Shell"
date: 2023-01-18T08:08:38-05:00
draft: false
tags: ['rds', 'aurora', 'mysql shell']
---

As probably with anyone experimenting with [Innodb Cluster](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-introduction.html) and [MySQL 8](https://dev.mysql.comdoc/refman/8.0/en/), I've spent a good deal of time with [MySQL Shell](https://dev.mysql.com/doc/mysql-shell/8.0/en/). I think for some old-timers MySQL Shell can initially come across as odd or difficult to understand.  Partially, I think it is because Shell offers a deluge of new features from:
* [managing local sandbox mysql instances](https://dev.mysql.com/doc/mysql-shell/8.0/en/admin-api-sandboxes.html)
* [3 different shell languages](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-active-language.html) (SQL, Javascript, and Python), 
* [full management APIs for Innodb Cluster](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-innodb-cluster.html), 
* [parallel logical backup and restore utilities](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-utilities-dump-instance-schema.html)
* and probably a whole lot more I haven't found yet.
  
However, as I've grown more comfortable with it, I can say that I do like it's possibilities.

## Credential storage

One of Shell's features is [credential storage](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-pluggable-password-store.html).  By default, you will be prompted if you want to save the password you use to make a successful mysql connection.  This is fine as a convenience, but what is more interesting to me is the fact that the credential store is pluggable.  But how?

The interface for creating a credential storage plugin is not documented (as far as I can tell) in the manual.  The best (and only) documentation I could find on this is from [a MySQL dev blog from 2018](https://dev.mysql.com/blog-archive/mysql-shell-8-0-12-storing-mysql-passwords-securely/), see the lower section 'Custom credential helpers'.

In a nutshell, a credential helper is just an executable that Shell runs on the same system you are running MySQL shell on.  The executable must implement 5 subcommands in the format `mysql-secret-store-yourname <subcommand>`.  Some subcommands take JSON on standard input, some emit JSON on standard output, some do both.  The executable should exit with a 0 on success or a 1 on failure for any of the comands.  The executable must be in a specific directory so that Shell can find it.  Simple enough, right?  The 5 subcommands are:
* store - stores credentials passed in
* get - gets requested credentials 
* erase - drop specific credentials from the store
* list - list all available credentials
* version - output a version number

## RDS and Aurora

How might I use the credential store?  Well, I am using RDS and Aurora more and more these days.  As a DBA type at my company, I need to access a wide variety of databases on AWS.  All of these have been setup to allow access to me as a DBA using a [standard IAM user](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html) using temporary IAM credentials.  The basic pre requisites are:
1. I have credentials to access a given AWS account with a specific (and limited) role that allows me to generate IAM tokens
2. There is a [mysql user setup connected to the role with a special plugin that associates it with IAM Authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.DBAccounts.html).

To get DB access I then:
1. I use my AWS credentials to generate an IAM token
2. I setup an SSH tunnel to a place where I can access the network addresses of the DB
3. I run my mysql client more or less as normal, but I have to pass the IAM token as my password and use the SSH tunnel port for proper routing


Using the AWS CLI and pseudo-bash, it would basically look like this:

```
ssh -L "$LOCAL_PORT:$ENDPOINT:$PORT" $SSH_BASTION &

TOKEN=`aws rds generate-db-auth-token \
  --hostname "$ENDPOINT" \
  --port "$PORT"\ 
  --profile "$AWS_PROFILE" \
  --username "$MYSQL_USER/IAM ROLE NAME"`

mysql -u "$MYSQL_USER/IAM ROLE NAME" "-p=$TOKEN" -h 127.0.0.1 -p $LOCAL_PORT
```

The token is good for 15 minutes, so I can cache the value for up to that long and subsequent connections can re-use it.  I can also re-use the SSH tunnel as long as it is open and connecting to the endpoint I want to use.

## Meeting the two in the middle

What if there was a way to connect these two systems?  I don't have a huge use case for storing static credentials, but what if I could write a Credential store that would handle fetching these temporary credentials for me?

It turns out that this is pretty easy. I can use any scripting or compiled language I want, provided I implement the needed interface and deal with Server URLs.

## Turning Server URLs into what we need
One big obstacle however is that the interface provides a single key: Server URL. This is comprised of:
* user name
* host name
* port number

Somehow I have to take that input and generate the following information so I can get my IAM token:
* the [AWS config profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) name for the target DB's account.  Other AWS credential types could be used here instead.  
* the AWS region of the specific host I'm connecting to
* the RDS/Aurora endpoint I'm connecting to (probably the hostname in the URL)
* the IAM role to get a token for (I believe this has to be the mysql user name, but I could be mistaken here) (probably the user name in the URL)

The solution for how to do this depends greatly on conventions very specific to my environment.  I ended up using a combination of parsing (i.e., to extract the region) and convention (my aws profile names and my RDS endpoints share common elements I can parse out).

 How you might need to implement this is going to be specific to your setup as well, but it could be as simple as a local YAML / JSON config file you manage by hand.  For example:

```yaml
my-endpoint.us-east-1.rds.amazonaws.com:
  aws_profile: profile_name
  region: us-east-1
```

Then you can take that profile and region and use it in your AWS API/CLI calls.  YMMV.  


## The interface

But how should the interface behave?  This isn't necessarily what the good folks at Oracle expected when they designed credential helpers.  Let's talk about how we might handle each subcommand:

###  get

This is the meat and potatoes where we need to turn the requested ServerURL into an IAM token.  We convert the given Server URL into our needed AWS information and use either the AWS API or CLI to generate a token.  

This method also handles caching the token for the 15 minutes.  If get is called again, we can use the cached version and skip all the AWS calls.  If the cache is expired, we fetch a new token as before.

This is a DB secret, so storage of this cache should be handled with some care.  

### store 

No-op.  We don't actually store credentials with this helper, just retrieve.

### erase

Shell calls this for some reason the authentication fails.  I implemented this to drop the Server URL in my cache in case the token needed to be regenerated, e.g., the token expired but my cache TTL didn't work.

### list

I simply list the Server URLs in my cache, i.e., my cache keys.  My cache doesn't actually drop entries until `get` is called, so this serves as a list of servers I've successfully authenticated to in the past (remember failed auth calls `erase` on a key).  I don't know if I will keep it that way or not yet.    

### version

Just emit some version string for my handler

## Handling the tunnel

It turns out that MySQL Shell already has [support for ssh tunneling](https://dev.mysql.com/doc/mysql-shell/8.0/en/mysql-shell-connection-ssh.html).  One less thing to worry about!  When I connect in the shell, I just give it the ssh bastion hostname I use above, and it does the rest.

## Tying it together

To use my credential helper, I have to have the executable be named `mysql-secret-store-<myname>` and it must be located in the same folder as the MySQL Shell binary.  I ended up writing my helper in Golang and when I have a new version I like to run `go install`, so I simply created a symlink from my go bin directory to where Homebrew installed Shell for me:
```sh
ln -sf $GOPATH/bin/mysql-secret-store-rds /usr/local/mysql-shell/bin/mysql-secret-store-rds
```

I have to then update my shell's config to use this helper
```
MySQL  JS > \option --persist credentialStore.helper='rds'
```

You may or may not want this to be a permanent setting.  I am planning on making mine permanent, but extending my handler to be a transparent wrapper around another handler when I determine the Server URL is not RDS/Aurora.

Finally, I'm ready to make a connection:

```
MySQL  JS > shell.connect({
 	ssh:"bastion.host.name",
 	host:"rds.endpoint.rds.amazonaws.com",
 	port: "3306",
 	user: "dba-user",
 	"ssl-ca":"/Users/jayj/.aws/rds-combined-ca-bundle.pem",
 	"ssl-mode": "VERIFY_CA"
})
Creating a session to 'dba-user@rds.endpoint.rds.amazonaws.com:3306?ssl-ca=%2FUsers%2Fjayj%2F.aws%2Frds-combined-ca-bundle.pem&ssl-mode=VERIFY_CA'
Opening SSH tunnel to bastion.host.name:22...
Fetching schema names for auto-completion... Press ^C to stop.
Closing old connection...
Your MySQL connection id is 3692
Server version: 5.7.12-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
<ClassicSession:dba-user@rds.endpoint.rds.amazonaws.com:3306>
 MySQL  bastion.host.name > rds.endpoint.rds.amazonaws.com:3306 ssl  JS > \sql select @@aurora_version;
+------------------+
| @@aurora_version |
+------------------+
| 2.11.0           |
+------------------+
1 row in set (0.0445 sec)
```
Success!

Like I said before, I can't really open source my version of this because of the environment specific stuff I do to get from a Server URL to the AWS metadata I need to lookup the token.  I will say that my resulting Go code is only about ~300 lines and most of that is some semi-ugly string parsing.  My hope with this post is it inpires you to consider other ways MySQL Shell's pluggable credential helper might be useful for you.  



