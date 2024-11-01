
# Installing OpenVK on Fedora Server 40

  
 This is a sample guide for installing OpenVK on Fedora Server 40, created by user bulbad0za.
  

## SELinux

  

🖥Run the command:

  

```bash

getenforce

```

  

If it says `Enforcing` then SELinux will disturb us. Let's disable it.

  

!!! note

I know that it's not most secured solution but I don't know any proper way that will work.

  

📝Run command `setenforce 0` and check the SELinux status again with the `getenforce` command, it should now display `Permissive`.

  

## Dependencies

  

🖥Let's install Remi repos for PHP 8.2:

  

```bash

dnf install https://rpms.remirepo.net/fedora/remi-release-40.rpm

```

  

🖥Then enable modules that we need:

  

```bash

dnf -y module enable php:remi-8.2

```

  

🖥And install dependencies:

  

```bash

dnf -y install php php-cli php-common unzip php-zip php-yaml php-gd php-pdo_mysql nodejs git nano mariadb-server httpd

```

  

🖥Don't forget about Yarn and Composer:

  

```bash

npm i -g yarn

php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"

php composer-setup.php --filename=composer2 --install-dir=/bin --snapshot

```

  

### Database

  

🖥We will use MariaDB for DB:

  

```bash

systemctl start mariadb

systemctl enable mariadb

```

🖥Then run `mysql_secure_installation`, set new password and answer like this:

 Enter current password for root (enter for none): `*press enter*`
 
Change the root password? [Y/n] y

*You will then be asked to come up with a password and enter it again.*

Remove anonymous users? [Y/n] y

Disallow root login remotely? [Y/n] y


Reload privilege tables now? [Y/n] y

  

### ffmpeg

  

Additionally, you can install ffmpeg for processing videos.

  

🖥You will need to use RPMFusion repo to install it:

  

```bash

dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download1.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

```

  

🖥Then install ffmpeg:

  

```bash

dnf -y install ffmpeg

```

  

## Chandler and OpenVK installation

  

🖥Install Chandler in `/opt`:

  

```bash

cd /opt

git clone https://github.com/openvk/chandler.git

cd chandler/

composer2 install

```

  

🖥You will need a secret key. You can generate it using:

  

```bash

cat /dev/random | tr -dc 'a-z0-9' | fold -w 128 | head -n 1

```

  

📝Now edit config file `chandler-example.yml` like this:

  

```yaml title="/opt/chandler/chandler-example.yml" hl_lines="12-14 17"

chandler:

debug: true

websiteUrl: null

rootApp: "openvk"

preferences:

appendExtension: "xhtml"

adminUrl: "/chandlerd"

exposeChandler: true

database:

dsn: "mysql:unix_socket=/var/lib/mysql/mysql.sock;dbname=openvk"

user: "root"

password: "DATABASE_PASSWORD"

security:

secret: "SECRET_KEY_HERE"

csrfProtection: "permissive"

sessionDuration: 14

```

  

🖥And rename it to `chandler.yml`:

  

```bash

mv chandler-example.yml chandler.yml

```

  

🖥Now let's install CommitCaptcha extension. It is mandatory for OpenVK.

  

```bash

cd extensions/available/

git clone https://github.com/openvk/commitcaptcha.git

cd commitcaptcha/

composer2 install

```

  

🖥And now download OpenVK:

  

```bash

cd ..

git clone --recursive https://github.com/openvk/openvk.git

cd openvk/

composer2 install

cd Web/static/js

yarn install

cd /opt/chandler/extensions/available/openvk

```

  

📝Now edit config file `openvk-example.yml` like this:

  

```yaml

openvk:

debug: true

appearance:

name: "OpenVK"

motd: "Yet another OpenVK instance"

preferences:

femaleGenderPriority: true

uploads:

disableLargeUploads: false

mode: "basic"

shortcodes:

forbiddenNames:

- "index.php"

security:

requireEmail: false

requirePhone: false

forcePhoneVerification: false

forceEmailVerification: false

enableSu: true

rateLimits:

actions: 5

time: 20

maxViolations: 50

maxViolationsAge: 120

autoban: true

support:

supportName: "Moderator"

adminAccount: 1 # Change this ok

messages:

strict: false

wall:

postSizes:

maxSize: 60000

processingLimit: 3000

emojiProcessingLimit: 1000

menu:

links:

adPoster:

enable: false

src: "https://example.org/ad_poster.jpeg"

caption: "Ad caption"

link: "https://example.org/product.aspx?id=10&from=ovk"

bellsAndWhistles:

fartscroll: false

testLabel: false

credentials:

smsc:

enable: false

client: ""

secret: ""

eventDB:

enable: true # Better enable this

database:

dsn: "mysql:unix_socket=/var/lib/mysql/mysql.sock;dbname=openvk_eventdb"  #Note that you should use “openvk_eventdb” and not “openvk-eventdb”.

user: "root"

password: "DATABASE_PASSWORD"

```

  

Please note `eventDB` section because it's better to enable event database.

  

🖥And rename it to `openvk.yml`:

  

```bash

mv openvk-example.yml openvk.yml

```

  

🖥Then enable CommitCaptcha and OpenVK for Chandler:

  

```bash

ln -s /opt/chandler/extensions/available/commitcaptcha/ /opt/chandler/extensions/enabled/commitcaptcha

ln -s /opt/chandler/extensions/available/openvk/ /opt/chandler/extensions/enabled/openvk

```

  

### DB configuration

  

!!! note

It's better to create another user for SQL but I won't cover that.

  

🖥Enter MySQL shell:

  

```bash

mysql -p

```

  

🖥And create main and event databases:

  

```sql

CREATE DATABASE openvk;

CREATE DATABASE openvk_eventdb;

exit

```

  

🖥Go to `/opt/chandler`:

  

```bash

cd /opt/chandler

```

  

📝We need to import Chandler database :


  

🖥Now database dump can be imported:

  

```bash

mysql -p'DATABASE_PASSWORD' openvk < install/init-db.sql

```

  

🖥Go to `extensions/available/openvk/`:

  

```bash

cd extensions/available/openvk/

```

  

📝We also need to import OpenVK database:


  

🖥Now database dump can be imported:

  

```bash

mysql -p'DATABASE_PASSWORD' openvk < install/init-static-db.sql

```

  

🖥Also import event database:

  

```bash

mysql -p'DATABASE_PASSWORD' openvk_eventdb < install/init-event-db.sql

```

🖥Now we need to go to the directory with all the OpenVK tables for the database:

    cd install/sqls
    ls

You will see all available databases that need to be imported one by one. You can use this template command:

    mysql -p'DATABASE_PASSWORD' openvk < *database table name*

Once you have imported everything you need to go to the eventdb directory and import one table:

    cd eventdb/
    mysql -p'DATABASE_PASSWORD' openvk_eventdb < 00000-notifications-emoji-support.sql

### Installing dependencies for PHP

We also need to install some dependencies for PHP for the instance to work correctly.

#### xdiff & imagick

    dnf install php-pear php-devel gcc ImageMagick ImageMagick-devel php-intl
    
    pecl install imagick
    
    cd /usr/
    curl -O http://www.xmailserver.org/libxdiff-0.23.tar.gz
    tar -xvf libxdiff-0.23.tar.gz    
    cd libxdiff-0.23/
    ./configure
    make
    make install
    pecl install xdiff

Once we have installed the required dependencies they need to be enabled in the `/etc/php.ini` file. Write it at the end of the file:

    extension=imagick.so
    extension=intl.so
    extension=xdiff.so

### Webserver configuration

  

Apache is already installed so we will use it.

  

🖥Make the user `apache` owner of the `chandler` folder:

  

```bash

cd /opt

chown -R apache: chandler/

```

  

📝Now let's create config file `/etc/httpd/conf.d/10-openvk.conf`:

  

```apache

<VirtualHost *:80>

ServerName openvk.local

DocumentRoot /opt/chandler/htdocs

  

<Directory /opt/chandler/htdocs>

AllowOverride All

  

Require all granted

</Directory>

  

ErrorLog /var/log/openvk/error.log

CustomLog /var/log/openvk/access.log combinedio

</VirtualHost>

```

  

📝Also enable rewrite_module by creating `/etc/httpd/conf.modules.d/02-rewrite.conf`:

  

```apache

LoadModule rewrite_module modules/mod_rewrite.so

```

  

🖥Make directory for OpenVK logs and make the user `apache` owner of it:

  

```bash

mkdir /var/log/openvk

chown apache: /var/log/openvk/

```

  

🖥Make the firewall exception for port 80:

  

```bash

firewall-cmd --permanent --add-port=80/tcp

firewall-cmd --reload

```

  

🖥And start Apache:

  

```bash

systemctl start httpd

```

  


Congratulations, OpenVK successfully installed! But there are still some bugs that can be fixed via phpMyAdmin (how to install and configure it I think you can figure it out yourself). The bugs and their solutions will be described below:

### Adding new administrators
Since new admins cannot be added to OpenVK through the normal admin panel we will do it through phpMyAdmin. 

Open the openvk database and open the `ChandlerACLRelations` table, click on the `INSERT` tab. You will see 3 columns - `user`, `group` and `priority`. 

In `user` you must enter the UUID of the required user, it can be found in the admin panel in the Users tab.

In `group` you must enter the ID of the Chandler group, in our case it is the “Administrators” group. 

And in `priority` you just need to enter the value `64`. 

### Problems with using HTTPS

If you are using HTTPS, you may have a problem with playing music and other media files. This can be solved by editing the file located at `openvk/Web/Models/Entities/Media.php`. Go down to line 61 and edit it as follows:

    return "https://" . $_SERVER['HTTP_HOST'] . "/

Now you should have everything working, but note that if you want to use HTTP again, this line will need to be reverted.

### Broken gift sending/points transfers

After installing OpenVK and enabling commerce, you may not be able to give gifts or send points to other users. This can be fixed in the following way:

Open the `openvk_eventdb` database and in it the `notifications` table. Go to the `STRUCTURE` tab and click the Change button next to `modelAction`. Change the Type to `MEDIUMINT` and set the Value to `7`.

### Helpdesk is missing

For some reason the default account doesn't have access to the Helpdesk, let's fix that!
Open the `ChandlerACLGroupsPermissions` table and click the INSERT tab. Enter the Values in this order:

group - `3b55f3ca-2ccf-11ec-9487-fa163e1b15b1`
model - `openvk\Web\Models\Entities\TicketReply`
context - `0`
permission - `write`
status - `1`

After that you need to save everything and in `ChandlerACLRelations` add administrator to group `3b55f3ca-2ccf-11ec-9487-fa163e1b15b1` (in general actions are almost identical as with adding administrator).
After these steps, Helpdesk should appear in your sidebar.
