# Virtual Exim 2
## README AND INSTALL GUIDE

Thanks for picking the Virtual Exim package, for your virtual mail hosting needs! :-)

This document provides a basic guide on how to get Virtual Exim working on your system. In this guide, I assume that you have *a little* knowledge of both MySQL/PostgreSQL and Exim.

Before we go into any details, I'd like to thanks Philip Hazel and the Exim developers for a fine product. I would also like to thanks the postmasters at various domains for letting me play havoc with their mail while I set this up :-) Finally, a special note of thanks to Dan Bernstein for his Qmail MTA. Dan, thank you for educating me how mail delivery really shouldn't be done, on the Internet.

The Virtual Exim project currently lives on GitHub: https://github.com/vexim/vexim2
And its mailing list/Google group is available at: https://groups.google.com/group/vexim

## Installation steps for each component:

##### NOTE FOR UPGRADING:
If you are upgrading from a previous version of Virtual Exim, you'll find additional notes marked 'UPGRADING' in some sections. If and when you do, follow these notes.

##### DISTRIBUTION-SPECIFIC NOTES:
Some sections may contain distribution or OS-specific notes. You'll find them after an appropriate prefix, such as 'DEBIAN' or 'FREEBSD' where appropriate.

## PARTS:
1. [Prerequisites](#prerequisites)
2. [System user](#system-user)
3. [Databases and authentication](#databases-and-authentication)
4. [Files and Apache](#files-and-apache)
5. [Mailman](#mailman)
6. [Exim configuration](#exim-configuration)
7. [Site Admin](#site-admin)
8. [Virtual Domains](#virtual-domains)
9. [Mail storage and Delivery](#mail-storage-and-delivery)
10. [POP3 and IMAP daemons (separate to this software)](#pop3-and-imap-daemons)
11. [SIEVE support considerations](#sieve_considerations)
12. [NGINX instead of Apache](#nginx_webserver)

## Prerequisites:
The following packages must be installed on your system, for Virtual Exim to work. If you don't have any of these packages already installed, please refer to the documentation provided with your operating system on how to install each package:
* Exim v4 with MySQL or PostgreSQL support (tested on v4.1x/4.2x/4.7x)
* MySQL (tested on v5.1.x) or PostgreSQL
* Apache or other HTTP server (Tested on Apache v2.2.x) (NGINX now also works. See chapter 12)
* PHP (tested on v5.3.x and v7.x) with at least the following extensions:
  * PDO
  * pdo_mysql or pdo_pgsql
  * imap
  * gettext
  * iconv
  * filter

The following packages provide optional functionality:
* Mailman – to have mailing lists
* ClamAV – for scanning e-mail for viruses
* SpamAssassin – for scanning e-mail from spam

VExim might work with older (or newer) versions of these packages, but you may have to perform some adaptation work to achieve that. In any case, you are welcome to file bugs and/or provide patches on GitHub.

**DEBIAN with APACHE:** The following command line installs all the packages mentioned above (last four are optional), if you're going with MySQL setup:
```
# apt-get install apache2 exim4-daemon-heavy mysql-server libapache2-mod-php5 php5-mysql php5-imap clamav-daemon clamav-freshclam spamassassin mailman
```

The PostgreSQL setup would use something like this:
```
# apt-get install apache2 exim4-daemon-heavy postgresql postgresql-contrib libapache2-mod-php5 php5-pgsql php5-imap clamav-daemon clamav-freshclam spamassassin mailman 
```

For NGINX, see chapter 12 to see what you should do differently.

## System user:
You should create a new user account to whom the virtual mailboxes will belong. Since you do not want anyone to be able to login using that account, you should also disable logging in for that user. Here are the command lines to do that. This manual assumes you want to have your virtual mailboxes in /var/vmail. If you want them elsewhere, adjust the commands. After the user and group are created, find their uid and gid using the last command and memorize these values:
```
# useradd -r -m -U -s /bin/false -d /var/vmail vexim
# id vexim
```
**FREEBSD:** Instead of the commands above, you should probably use the following (change `90` to another value if this user or group id is already used on your system):
```
# pw groupadd vexim -g 90
# pw useradd vexim -u 90 -g vexim -d /usr/local/mail -m -s /nonexistant
```
**DEBIAN:** Use the following command instead:
```
# adduser --system --home /var/vmail --disabled-password --disabled-login --group vexim
```

## Databases and authentication:

### MySQL:
This distribution contains a file "vexim2/setup/mysql.sql". This file provides the database schema used by vexim. You will have to import it into MySQL, like this:
```
# mysql -u root -D YOUR_DATABASE_NAME -p < vexim2/setup/mysql.sql
```
Where `YOUR_DATABASE_NAME` is the name of an empty database you have created for vexim. If you want the script to create the database for you and set up access to it, this is also doable: just open it in a text editor, and find a commented out block which begins with `-- CREATE DATABASE` near the top of the file. This block is documented just above it, so you may uncomment it, apply the changes you want and save the file. With the necessary changes made, you should run the following command line to initialize the database:
```
# mysql -u root -p < vexim2/setup/mysql.sql
```
A site admininistrator account is created with an autogenerated password required for your first login to Vexim.

### PGSQL:
The code has been tested by several users to work with Virtual Exim, and we try our best to make sure it always will. Unfortunately I don't have much PostgreSQL knowledge to support it fully. A database schema for it is included however, as setup/pgsql.sql to help you set up the database. Make sure to adjust it similarly as per MySQL instructions above.

**DEBIAN:**
When using Postgres and Debian, you will note that it is preferred practice to not do datababase-manipulations as root, but instead use the specially available 'postgres' user to do these kinds of things with. Simply su/sudo to user 'postgres' and use something like this to import/run the SQL-file for postgres after making any neccesary changes regarding passwords. At the top of the file, it contains the SQL-statements to create the database, commented out. Edit these to your liking and uncomment them. Please notice the info about using postgresql-9.6 and upwards (NOCREATEUSER not supported):
```
# su - postgres
$ psql < pgsql.sql
```

If you create a user/database by hand, simply comment out the database-creation above and use:
```
$ psql -U <dbuser> -W <dbname> < pgsql.sql
```
This will ask for the password you used to create the 'dbuser' and then run the statements in pgsql.sql

Notes: 
Ignore any 'NOTICE' messages that inform you about tables not existing and skipping; these are a result of the SQL-file making sure that any existing tables be deleted before creating them to avoid errors later on.

If you see errors about not being able to enable the pgcrypto extension, be sure to check if you have pgsql-contrib installed.
 

**UPGRADING:**
If you are upgrading your installation, we have prepared MySQL migration scripts for you, which you will find under vexim2/setup/migrations/. Find out the version of Vexim that you have and apply the necessary scripts in a sequential manner, like this:
```
# mysql -u root -D YOUR_DATABASE_NAME -p < vexim2/setup/migrations/SCRIPT_FILENAME.sql
```

At current, there are no PostgreSQL equivalents of the mysql upgrade-scripts. These might be included in a next release.

## Files and Apache:
In this distribution is a directory called 'vexim'. You have two options:
* Copy this directory into your current DocumentRoot for your domain, and optionally rename the directory.
* Set up a new VirtualHost and point the DocumentRoot to the vexim directory.

Both should work equally well.
(Note: a chapter is provided for basic information on how to host vexim on NGINX instead of Apache. Simply ignore the two options listed above and continue below).


After copying the 'vexim' directory, you should find the 'variables.php.example', file in its subdirectory called 'config', copy that file to 'variables.php' and change the following values defined in it:
* $sqlpass – to the vexim database user's password which you chose while editing 'mysql.sql' in the "Databases and authentication" step.
* $uid, $gid and $mailroot to the values you have from the "System user" step.
* $cryptscheme is set to "sha512", a more specific configuration or other crypt-schemes can be used.
* $mailmanroot to the mailman URL

Other, less interesting options are documented in the comments of that file. Feel free to explore them as well.

## Mailman:
Mailman needs to be installed if you want to use mailing lists. Edit the default configuration file (`/etc/mailman/mm_cfg.py`):
```
DEFAULT_URL_PATTERN = 'https://%s/mailman/'
DEFAULT_SERVER_LANGUAGE = 'en'
DEFAULT_EMAIL_HOST = 'mail.example.tld'
DEFAULT_URL_HOST   = 'mail.example.tld'
```
Debian will already create a default configuration for your webserver that you can enable with `a2ensite mailman`. Create your master password: `mmsitepass MY_PASSWORD``. Restart mailman and apache.

## Exim configuration:
**NOTE:** the configuration files supplied here have been revised. You should use them carefully and report problems!

An set of example Exim 'configure' files, has been included with this distribution as 'docs/configure.mysql' and 'docs/configure.pgsql'. Copy the correct one to the location Exim expects its configuration file to be on your installation. You will also need to copy docs/vexim* to /usr/local/etc/exim/. The following lines are important and will have to be edited if you are using this configure, or copied to your own configure file:


Edit these if your mailman is in a different location (in Debian: `/var/lib/mailman`):
```
MAILMAN_HOME=/usr/local/mailman
MAILMAN_WRAP=MAILMAN_HOME/mail/mailman
```
These need to match the username and group under which exim runs (in Debian: `list`/`daemon`):
```
MAILMAN_USER=mailnull
MAILMAN_GROUP=mail
```
Change this to the name of your server:
```
primary_hostname=mail.example.org
```
In general, it is required that your reverse DNS entry of your IP points to this hostname.

Depending on the database type you are using, you will need to uncomment the appropriate lines in the config, to enable lookups. The explanation below uses the 'configure.mysql' file as an example. The 'configure.pgsql' file contains the same type of config that requires the same changes.

These control which domains you accept mail for and deliver locally (local_domains), which domains you accept mail for and deliver remotely (relay_to_domains), which IP addresses are allowed to send mail to any domain (relay_from_hosts) and which system users are considered trusted (trusted_users). More on these options – in Exim documentation.
```
domainlist local_domains = @ : example.org : ${lookup mysql{VIRTUAL_DOMAINS}} : ${lookup mysql{ALIAS_DOMAINS}}
domainlist relay_to_domains = ${lookup mysql{RELAY_DOMAINS}}
hostlist   relay_from_hosts = localhost : @ : 192.168.0.0/24
#trusted_users = www-data
```
These lines configure database connectivity. You need to uncomment one of them (depending on the database type you have chosen) and adjust it to match your setup. You at least have to change the word 'CHANGE' to the password you used for the 'vexim' database user, which you have created before. The socket path depends on your system, for Debian it is:
```
#hide mysql_servers = localhost::(/var/run/mysqld/mysqld.sock)/db/user/password
#hide pgsql_servers = (/var/run/postgresql/.s.PGSQL.5432)/db/user/password
```
If you want to use either Anti-Virus scanning, or SpamAssassin, you will need to uncomment the appropriate line here.
```
# av_scanner = clamd:/tmp/clamd
# spamd_address = 127.0.0.1 783
```
in Debian use:
```
# av_scanner = clamd:/var/run/clamav/clamd.ctl
# spamd_address = /var/run/spamd.sock
```
Specify here, the username and group under which Exim runs (Debian: `Debian-exim`). This combination is also that under which mailman must run in order to work:
```
exim_user = mailnull
exim_group = mail
```
Also it is assumed that the mysql domain socket is /tmp/mysql.sock, which is where the FreeBSD port puts it. Other installations put it in /var/tmp, /usr/lib, or any number of other places. If yours isn't /tmp/mysql.sock, you will need to set this. 

TLS is activated by default. We suppose that you already created a SSL key and certificate.
```
tls_certificate = /etc/exim4/exim.crt
tls_privatekey = /etc/exim4/exim.key
```
The creation of SSL-keys is the same like for webservers, e.g. https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-with-a-free-signed-ssl-certificate-on-a-vps . You can use the same certificate your the webserver of this host (if you use webmail).

```
tls_dhparam = /etc/exim4/dhparam.pem
```
The Diffie-Hellman group should have at least 1024 bit and can be created with this command (it can take some time):
```
# openssl dhparam -out /etc/exim4/dhparam.pem 2048
```
In `tls_require_ciphers`, currently (2016) secure ciphers are selected. It works by default on GnuTLS setups (Debian/Ubuntu). If your distribution uses OpenSSL (e.g. FreeBSD, CentOS), comment the block `tls_require_ciphers = ...` and uncomment the line `openssl_options = ...`. If you are not sure, the output of `exim -bV` will show either GnuTLS or OpenSSL.


###### ACL's:
We have split all of the ACL's into separate files, to make managing them easier. Please review the ACL section of the configure file. If there are ACL's you would rather not have executed, please comment out the '.include' line that references them, or edit the ACL file directly and comment them out.

###### DEBIAN:
Typically, Debian setups use split Exim configuration with some Debconf magic. This manual will assume that you are familiar with it. If not, you should refer to the Debian documentation on Exim. The gist of it is that Debian allows managing the Exim configuration file through the use of a /etc/exim4/conf.d/ directory, merging files together into a single 'normal' config file after running 'update-exim4.conf'. It takes a number of settings from the file /etc/exim4/update-exim4.conf.conf too, making it easy to change some simple settings without having to hunt for the exact spots these things get configured at in Exim.

As such, Vexim2 includes a set of config-files for mysql and for PostgreSQL under docs/debian-conf.d/mysql/ and docs/debian-conf.d/pgsql/ that can be used to simply merge into the /etc/exim4/conf.d hierarchy.

 To get the virtual mailboxes to work, copy either the docs/debian-conf.d/mysql or docs/debian-conf.d/pgsql files into /etc/exim4/conf.d  and change the database password in .../main/00_vexim_listmacrosdefs. You may also want to review the ACL's in docs/vexim-acl-*.conf and selectively copy and paste their contents to the files provided by Debian in conf.d. By the way, some of these ACL's are already implemented by Debian, so you might just need to enable them by defining certain macros as described in Debian manual. This manual does not cover enabling ClamAV and SpamAssassin in Exim in Debian. Please look this up elsewhere. By the way, the author of this part never bothered to set up Vexim in such a way that Debian would take into account the status of the various user flag (on_av, on_spamassassin etc) for each user. In his setup, these flags have no effect, and all messages are checked for spam and viruses.

Stefan Tomanek has a nice writeup about using Vexim in Debian, but that article does not cover all aspects, is a bit outdated, and most of if has been incorporated (and improved!) into this document anyway. You can find it at http://stefans.datenbruch.de/rootserver/vexim.shtml.


## Site Admin:
In order to add and delete domains from the database, you need to have a "site admin". This user can create the initial postmaster users for the individual domains. This user has been created along with the database (see 'databases and authentication'-section), use it here to log in. The password is case sensitive. You are advised to change it when you first log in.

## Virtual Domains:
Virtual Exim can now control which local domains Exim accepts mail for and which domains it relays mail for. The features are controlled by the siteadmin, and domains can be easily added/removed from the siteadmin pages. Local domains can also be enabled/disabled on the fly, but relay domains are always enabled.


## Mail storage and Delivery:
The vexim configuration assumes that mail will be stored in /var/vmail/domain.com/username/Maildir. If you want to change the path from '/var/vmail/', you need to edit the file:
>vexim/config/variables.php

and change 'mailroot' to the correct path. Don't forget the / at the end.

## POP3 and IMAP daemons:
There are many POP3 and IMAP daemons available. Some that we have found that work are:

* **Courier:** docs/clients/courierimap.txt
* **Dovecot:** docs/clients/dovecot.txt

Dovecot provides more features (server-side sieve filters) and is more performant on larger setups.
For the purpose of setting up 'ltmp' support (required for sieve filtering to work), extra documentation is provided in 'dovecot-lmtp.txt' Required only if you want to enable sieve support (see below)

**UPGRADING:** If you are upgrading, you will need to update your configs for your POP/IMAP daemons, as the database layout has changed. You should be able to follow the above instructions without problem.

## Sieve Consideration

When accessing a mailbox via multiple mail-clients (web-based, desktop, mobile), it soon becomes a problem that mail-filtering "traditionally" is handled on the mail-client unless your "ISP" handles server-side filtering for you. Typically, this would be done for SPAM-filtering, but being able to auto-sort your mail into multiple folders is something that server-side filtering enables.

This is where 'sieve' comes in. It is essentially an interpretor for a so-called 'sieve filters' that's evaluated each time a mail is presented to a particular sieve-enabled mailbox. 

For more info on sieve, please refer to 'https://sieve.info' as a full guide on how to use it is beyond the scope of this guide, though there might be some more links to documentation presented below that newcomers to sieve can benefit from.

To enable sieve to work, there are a number of changes required in the delivery-flow compared to 'stock vexim'.
The changes provided are inspired on the work of vexim co-developer 'runout-at' as listed on https://github.com/vexim/vexim2/pull/247

The main thing to understand about the changes is that  they involve routing the mail through Dovecot upon delivery of a message. This is because Dovecot is the program with the Sieve functionality , as well as the 'Manage-Sieve' functionality to manage/upload filters. It does so through the 'Local Mail Transport Protocol', aka LMTP. So to get SIEVE, we need to get EXIM to deliver mail to Dovecot via LMTP; requiring changes on both the side of EXIM and Dovecot.

For convenience, the files that are different for lmtp-support in EXIM are provided in 'docs/debian.conf.d/mysql-lmtp' and 'docs/debian.conf.d/pgsql-lmtp'. Simply take the relevant version (mysql or pgsql) *AFTER* having copied over everything from the regular (non-lmtp) configuration-hierarchy.
If you are NOT using debian or have NOT split the configuration into small files, note that the 'configure.pgsql' and 'configure.mysql' files do not currently have an lmtp-enabled version available. This is a TODO, but changes from the seperate config-files can be merged in by hand by people willing to do so.

For Dovecot, there is the text-file  'docs/clients/dovecot-lmtp.txt' that details all the EXTRA configuration required to get LMTP/SIEVE support configured on Dovecot. Note that the 'docs/clients/dovecot.txt' instructions should be followed (and tested) before proceding to enabling LMTP.


As for documentation, you can find some great info and/or tools in the following places:

- https://sieve.info
- https://wiki.dovecot/Pigeonhole/Sieve
- https://www.fastmail.com/help/technical/sieve-examples.html
- https://addons.thunderbird.net/en-US/thunderbird/addon/sieve/

## NGINX webserver

Getting NGINX to be your web-frontend instead of Apache is not too difficult a thing to do at all.

On debian, you will require PHP-fpm (php7.0-fpm works fine), and a php module for your database of choice. The example below here assumes postgres, but you can use 'php7.0-mysql' instead. Of course, we also require nginx if not installed already:

 # apt-get install php7.0-fpm php7.0-pgsql nginx-full

After the basics are installed, you can set up an http-server for a domain in the usual way and add some configuration to get php to work and a /vexim directory to serve you the vexim configuration website. 

Let's assume we want to make the site work as follows:

 - URL: http://mailserver.maildomain.example' (NO https. This is beyond the scope of this guide but SHOULD be added)
 - Site root-dir : /var/www/mailserver.maildomain.example/htdocs/'


/etc/nginx/sites-available/mailserver.maildomain.example.conf:

server {
  listen *:80;

  server_name	mailserver.maildomain.example;

  root /var/www/mailserver.maildomain.example/htdocs;
  autoindex off;
  access_log /var/log/nginx/mailserver.maildomain.example.access.log combined;
  error_log /var/log/nginx/mailserver.maildomain.example.error.log;

  location / {
    try_files $uri $uri/ =404;
  }
  location ~ ^/vexim/.*\.php$ {
    include snippets/fastcti-php.conf
    fastcgi_pass unix:/var/run/php/php7.0-fpm.sock;
  }
  location ~/\.ht {
    deny all;
  }
}

Make sure to put the contents of the 'vexim' directory in the vexim2 distribution into the directory referred to in the configuration file: '/var/www/mailserver.maildomain.example/htdocs/vexim'

At this point, you should be able to restart the nginx webserver and have it all working. Note that HTTPS/SSL support is *GREATLY* encouraged. Since there are multiple approaches to attaining an SSL-cert, it is not attempted to cover this aspect here. Using letsencrypt certificates for SMTP/IMAPS/MangeSieve and HTTP(s) is greatly encouraged, of course.

