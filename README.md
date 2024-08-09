# nextcloudSetup

## Nextcloud installation 
####  Install PHP and PHP extensions for Nextcloud
`$ sudo apt install php php-cli php-fpm php-json php-common php-mysql php-zip php-gd php-intl php-curl php-xml php-mbstring php-bcmath php-gmp`

- check php version
  `$ php -v`
  ```
   PHP 8.2.20 (cli) (built: Jun 17 2024 13:33:14) (NTS)
   Copyright (c) The PHP Group
   Zend Engine v4.2.20, Copyright (c) Zend Technologies
   with Zend OPcache v8.2.20, Copyright (c), by Zend Technologies
```

 - After installing all the packages, edit the php.ini file:
   ` $ sudo vi /etc/php/8.2/fpm/php.ini`

- Change the following settings per your requirements:

```
   max_execution_time = 300
   memory_limit = 512M 
   post_max_size = 128M 
   upload_max_filesize = 128M
```

- To implement the changes, restart the php-fpm service

  `$ sudo systemctl restart php8.2-fpm`

#### Install and configure PostgreSQL

`$ sudo apt install postgresql postgresql-contrib postgresql-client php-pgsql`
- check version after installation first switch current user to postgres
  `sudo -i -u postgres`
  `$ psql -version`
- out put will be:
  `psql (PostgreSQL) 16.3 (Debian 16.3-1.pgdg120+1)`

  ## Create a PostgreSQL Database and User:
- From inside psql shell
    1.Create a new user:

  `CREATE USER nextclouduser WITH PASSWORD '1234';`

2.Create a new database owned by this user
 
 `CREATE DATABASE nextclouddb OWNER nextclouduser;`

3.Grant all privileges to the new user on the database:

`GRANT ALL PRIVILEGES ON DATABASE nextclouddb TO nextclouduser;`

- From outside psql shell

  - Create first the user for the database.
      nextclouddb
       - first add postgres in sudoers group
       `  $ usermod -aG sudo postgres`

 `$ sudo -u postgres psql -c "CREATE USER nextcloud_user PASSWORD 'nextcloud_pw';"`

- And now create the database.

`$ sudo -u postgres psql -c "CREATE DATABASE nextcloud_db WITH OWNER nextcloud_user ENCODING=UTF8;"`

- Check if we now can connect to the database server and the database in detail (you will get a question about the password for the database user!). If this is not working it makes no sense to proceed further! We need to fix first the access then!

`$ psql -h localhost -U nextcloud -d nextcloud_db`

               or: 
    
`$ psql -U nextcloud_user -d nextcloud_db -h 127.0.0.1 -W`

 - Come out from data base using \q

####  Download and Install Nextcloud

- Use the following command to download the latest version of Nextcloud:
   `$ wget  https://download.nextcloud.com/server/releases/latest.zip `

 - Extract file into the folder /var/www/ with the following command:
    `$ sudo unzip latest.zip -d /var/www/`


    
