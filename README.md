# NextcloudSetup

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
   `$ sudo vi /etc/php/8.2/fpm/php.ini`

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
      
       - first add postgres in sudoers group

    `$ usermod -aG sudo postgres`

    `$ sudo -u postgres psql -c "CREATE USER nextcloud_user PASSWORD 'nextcloud_pw';"`

- And now create the database.
 
     `$ sudo -u postgres psql -c "CREATE DATABASE nextcloud_db WITH OWNER nextcloud_user ENCODING=UTF8;"`

- Check if we now can connect to the database server and the database in detail (you will get a question about the password for the database user!). If this is not working it makes no sense to proceed further! We need to fix first the access then!

     `$ psql -h localhost -U nextcloud -d nextcloud_db`

    
     `$ psql -U nextcloud_user -d nextcloud_db -h 127.0.0.1 -W`

 - Come out from data base using \q

####  Download and Install Nextcloud

- Use the following command to download the latest version of Nextcloud:

    `$ wget  https://download.nextcloud.com/server/releases/latest.zip `

- Extract file into the folder /var/www/ with the following command:

    `$ sudo unzip latest.zip -d /var/www/`

- Change ownership of the /var/www/nextcloud directory to www-data.

`$ sudo chown -R www-data:www-data /var/www/nextcloud`

#### Configure Nginx for Nextcloud with self signed certified

 -Generate the private key and certificate:
  
  `$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nextcloud.key -out nextcloud.crt`
 
  `$ sudo cp nextcloud.crt /etc/ssl/certs/ cp nextcloud.key /etc/ssl/private/`
 
  - Change nginx configuration
    
     `$ sudo vi /etc/nginx/sites-available/nextcloud.conf`

 `$ sudo vi /etc/nginx/sites-available/nextcloud.conf`
 
 - Add snippet inside file  and save it

```
upstream php-handler {
    server unix:/run/php/php8.2-fpm.sock;  # Adjust based on your PHP version
    # server 127.0.0.1:9000;  # Use if PHP-FPM is listening on a TCP port
}
server {
    listen 80;
    server_name nextcloud.local;
    # Enforce HTTPS
    return 301 https://$server_name$request_uri;
}
server {
    listen 443 ssl;
    server_name nextcloud.local;
    ssl_certificate /etc/ssl/certs/nextcloud.crt;
    ssl_certificate_key /etc/ssl/private/nextcloud.key;
    # Add headers to serve security-related headers
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header X-Download-Options noopen;
    add_header X-Permitted-Cross-Domain-Policies none;
    add_header Referrer-Policy no-referrer;
    # Path to the root of your installation
    root /var/www/nextcloud;
    index index.php index.html;
    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }
    # The following 2 rules are only needed for the user_webfinger app.
    # Uncomment it if you're planning to use this app.
    #rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
    #rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;
    location = /.well-known/carddav {
        return 301 $scheme://$host/remote.php/dav;
    }
    location = /.well-known/caldav {
        return 301 $scheme://$host/remote.php/dav;
    }
    location /.well-known/acme-challenge { }
    location ^~ / {
        # set max upload size
        client_max_body_size 512M;
        fastcgi_buffers 64 4K;
        # Enable gzip but do not remove ETag headers
        gzip on;
        gzip_vary on;
        gzip_comp_level 4;
        gzip_min_length 256;
        gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
        gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/otf font/ttf image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;
        location / {
            rewrite ^ /index.php;
        }
        location ~ ^\/(?:build|tests|config|lib|3rdparty|templates|data)\/ {
            deny all;
        }
        location ~ ^\/(?:\.|autotest|occ|issue|indie|db_|console) {
            deny all;
        }
        location ~ \.php(?:$|/) {
            fastcgi_split_path_info ^(.+?\.php)(/.*)$;
            set $path_info $fastcgi_path_info;
            try_files $fastcgi_script_name =404;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $path_info;
            fastcgi_param HTTPS on;
            fastcgi_param modHeadersAvailable true; #Avoid sending the security headers twice
            fastcgi_pass php-handler;
            fastcgi_intercept_errors on;
            fastcgi_request_buffering off;
        }
        location ~ ^\/(?:updater|ocs-provider)\/ {
            try_files $uri/ =404;
            index index.php;
        }
        location ~* \.(?:css|js|woff|svg|gif)$ {
            try_files $uri /index.php$uri$is_args$args;
            access_log off;
            expires 30d;
            add_header Cache-Control "public, max-age=15778463";
        }
        location ~* \.(?:png|html|ttf|ico|jpg|jpeg)$ {
            try_files $uri /index.php$uri$is_args$args;
            access_log off;
            expires 30d;
            add_header Cache-Control "public, max-age=15778463";
        }
    }
}
```
  - Symlink  site available to site enabled

    `$ ln -s /etc/nginx/sites-available/nextcloud.conf /etc/nginx/sites-enabled/`

 -  Restart nginx and access on browser through server name

```
    Database user: nextclouduser
    Database password: The password you set for nextclouduser (1234)
    Database name: nextclouddb
    Database host: localhost (or your PostgreSQL server address)
     
```
- Sample Commands for File Permissions



-  Move the data directory outside the web server document root and change it's permission
  
    `$ sudo mv /var/www/html/nextcloud/data /var/nextcloud_data`
 
    `$ sudo chown -R www-data:www-data /var/nextcloud_data`

    `$ chown -R www-data:www-data /var/www/nextcloud/`

-  Update Nextcloud Configuration:
 
  1. Open the config/config.php file of your Nextcloud installation.

      `$ vi /var/www/nextcloud/config/config.php`

  3. Update the 'datadirectory' parameter to point to the new location of your data directory.
   
     ```  'datadirectory' => '/var/nextcloud_data' ```

 
-  Restart Services:

      `$ sudo systemctl start nginx`
 
  - Test nginx using, if all things fine with syntax go further

      `$ sudo nginx -t`


- Now nextcloud is ready to use, access it from browser

  
### Error might occur

1. .ocdata is not found inside data directory
    
    - Create file using touch and give necessary permission

         `sudo touch /var/nextcloud_data/.ocdata`
      
         `sudo chown -R www-data:www-data /path/to/your/data`



## If you forget admin password of nextcloud how to get it back using postgres database. 
 
 1. Log in to your server:
    - SSH into the server where your PostgreSQL database is hosted.

 2. Switch to the PostgreSQL user:
    - `$ sudo -i -u postgres`
 
 3. Access the PostgreSQL command line
    - `psql`

 4. List the databases:(If you're unsure which database is being used by Nextcloud, you can list all the databases)
    - `\l`
 
 5. Connect to the Nextcloud database:
     - Connect to the specific database that Nextcloud is using.
     - `\c nextclouddb`

 6. Reset the password for the Nextcloud database user:
     - `ALTER USER nextcloud_user WITH PASSWORD 'new_password';`

 7. Exit the PostgreSQL command line:
     - `\q`

 8. Verify Database Configuration

    - Check the database connection details in the config.php file to ensure they are correct
       `sudo vi /var/www/nextcloud/config/config.php`

    - Replace nextcloud_db, nextcloud_user, and your_password with your actual database name, user, and password.
      ```
      'dbtype' => 'pgsql',
      'dbname' => 'nextcloud_db',
      'dbuser' => 'nextcloud_user',
      'dbpassword' => 'your_password',
      'dbhost' => 'localhost',
      'dbport' => '',

      ```

10. Restart nginx and access through browser.
