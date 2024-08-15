# Howto setup Nextcloud
This manual describes how to setup a Nextcloud a installation based on Nginx and PostgreSQL.

## Nextcloud installation

###  Install PHP and PHP extensions for Nextcloud
Installing the package `php(8.2)` would pull in the depending package `libapache2-mod-php8.2` which then would pull in also all apache2 depending packages. This is something we don't want to have so we using also `libapache2-mod-php8.2-` within the package list that is telling apt to ignore this package as an dependency.

`$ sudo apt install php php-cli php-fpm php-json php-common php-zip php-gd php-intl php-curl php-xml php-mbstring php-bcmath php-gmp php-pgsql libapache2-mod-php8.2-`

- Check php version (optional step)

  `$ php -v`

```
   PHP 8.2.20 (cli) (built: Jun 17 2024 13:33:14) (NTS)
   Copyright (c) The PHP Group
   Zend Engine v4.2.20, Copyright (c) Zend Technologies
   with Zend OPcache v8.2.20, Copyright (c), by Zend Technologies
```

 - After installing all the packages, edit the `php.ini` file:
   `$ sudo vi /etc/php/8.2/fpm/php.ini`

- Change the following settings per your requirements:

```
   max_execution_time = 300
   memory_limit = 512M 
   post_max_size = 128M 
   upload_max_filesize = 128M
```

- To make these settings effectiv, restart the php-fpm service

  `$ sudo systemctl restart php8.2-fpm`

### Install PostgreSQL and create a database and user

`$ sudo apt install postgresql postgresql-contrib postgresql-client`

- Check version after installation (optinal step)

  `sudo -i -u postgres`

  `$ psql -version`

- This out put will be seen:

   `psql (PostgreSQL) 16.3 (Debian 16.3-1.pgdg120+1)`

#### Create a PostgreSQL Database and User:

1. Create a new PostgreSQL user:

    `$ sudo -u postgres psql -c "CREATE USER nextcloud_user PASSWORD '1234';"`

3. Create new database and grant access:

    `$ sudo -u postgres psql -c "CREATE DATABASE nextcloud_db WITH OWNER nextcloud_user ENCODING=UTF8;"`

3. (Optionmal) Check if we now can connect to the database server and the database in detail (you will get a question about the password for the database user!). If this is not working it makes no sense to proceed further! We need to fix first the access then!

    `$ psql -h localhost -U nextcloud_user -d nextcloud_db`

    or

    `$ psql -U nextcloud_user -d nextcloud_db -h 127.0.0.1 -W`

- Log out from CLI base using the command `\q`

####  Download and install Nextcloud

- Use the following command to download the latest version of Nextcloud:

    `$ wget  https://download.nextcloud.com/server/releases/latest.zip `

- Extract file into the folder `/var/www/` with the following command:

    `$ sudo unzip latest.zip -d /var/www/`

- Change ownership of the `/var/www/nextcloud` directory to www-data.

    `$ sudo chown -R www-data:www-data /var/www/nextcloud`

#### Configure Nginx for Nextcloud with self signed certificate

- Generate the private key and certificate:

  `$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nextcloud.key -out nextcloud.crt`

  `$ sudo cp nextcloud.crt /etc/ssl/certs/ && sudo cp nextcloud.key /etc/ssl/private/`

- Change nginx configuration

  `$ sudo vi /etc/nginx/sites-available/nextcloud.conf`

 - Add the following snippet into the file and save it

```nginx
upstream php-handler {
    #server 127.0.0.1:9000;
    server unix:/run/php/php8.2-fpm.sock;
}

# Set the `immutable` cache control options only for assets with a cache busting `v` argument
map $arg_v $asset_immutable {
    "" "";
    default ", immutable";
}

server {
    listen 80;
    listen [::]:80;
    server_name nextcloud.local;

    # Prevent nginx HTTP Server Detection
    server_tokens off;

    # Enforce HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443      ssl http2;
    listen [::]:443 ssl http2;
    server_name nextcloud.local;

    # Path to the root of your installation
    root /var/www/nextcloud;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_certificate /etc/ssl/certs/nextcloud.crt;
    ssl_certificate_key /etc/ssl/private/nextcloud.key;

    # Prevent nginx HTTP Server Detection
    server_tokens off;

    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload" always;

    # set max upload size and increase upload timeout:
    client_max_body_size 512M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml text/javascript application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Pagespeed is not supported by Nextcloud, so if your server is built
    # with the `ngx_pagespeed` module, uncomment this line to disable it.
    #pagespeed off;

    # The settings allows you to optimize the HTTP2 bandwidth.
    # See https://blog.cloudflare.com/delivering-http-2-upload-speed-improvements/
    # for tuning hints
    client_body_buffer_size 512k;

    # HTTP response headers borrowed from Nextcloud `.htaccess`
    add_header Referrer-Policy                   "no-referrer"       always;
    add_header X-Content-Type-Options            "nosniff"           always;
    add_header X-Frame-Options                   "SAMEORIGIN"        always;
    add_header X-Permitted-Cross-Domain-Policies "none"              always;
    add_header X-Robots-Tag                      "noindex, nofollow" always;
    add_header X-XSS-Protection                  "1; mode=block"     always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Set .mjs and .wasm MIME types
    # Either include it in the default mime.types list
    # and include that list explicitly or add the file extension
    # only for Nextcloud like below:
    include mime.types;
    types {
        text/javascript js mjs;
	application/wasm wasm;
    }

    # Specify how to handle directories -- specifying `/index.php$request_uri`
    # here as the fallback means that Nginx always exhibits the desired behaviour
    # when a client requests a path that corresponds to a directory that exists
    # on the server. In particular, if that directory contains an index.php file,
    # that file is correctly served; if it doesn't, then the request is passed to
    # the front-end controller. This consistent behaviour means that we don't need
    # to specify custom rules for certain paths (e.g. images and other assets,
    # `/updater`, `/ocs-provider`), and thus
    # `try_files $uri $uri/ /index.php$request_uri`
    # always provides the desired behaviour.
    index index.php index.html /index.php$request_uri;

    # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Make a regex exception for `/.well-known` so that clients can still
    # access it despite the existence of the regex rule
    # `location ~ /(\.|autotest|...)` which would otherwise handle requests
    # for `/.well-known`.
    location ^~ /.well-known {
        # The rules in this block are an adaptation of the rules
        # in `.htaccess` that concern `/.well-known`.

        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }

        location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
        location /.well-known/pki-validation    { try_files $uri $uri/ =404; }

        # Let Nextcloud's API for `/.well-known` URIs handle all other
        # requests by passing them to the front-end controller.
        return 301 /index.php$request_uri;
    }

    # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

    # Ensure this block, which passes PHP files to the PHP process, is above the blocks
    # which handle static assets (as seen below). If this block is not declared first,
    # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
    # to the URI, resulting in a HTTP 500 error response.
    location ~ \.php(?:$|/) {
        # Required for legacy support
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|ocs-provider\/.+|.+\/richdocumentscode(_arm64)?\/proxy) /index.php$request_uri;

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;

        fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
        fastcgi_param front_controller_active true;     # Enable pretty urls
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;

        fastcgi_max_temp_file_size 0;
    }

    # Serve static files
    location ~ \.(?:css|js|mjs|svg|gif|png|jpg|ico|wasm|tflite|map|ogg|flac)$ {
        try_files $uri /index.php$request_uri;
        # HTTP response headers borrowed from Nextcloud `.htaccess`
        add_header Cache-Control                     "public, max-age=15778463$asset_immutable";
        add_header Referrer-Policy                   "no-referrer"       always;
        add_header X-Content-Type-Options            "nosniff"           always;
        add_header X-Frame-Options                   "SAMEORIGIN"        always;
        add_header X-Permitted-Cross-Domain-Policies "none"              always;
        add_header X-Robots-Tag                      "noindex, nofollow" always;
        add_header X-XSS-Protection                  "1; mode=block"     always;
        access_log off;     # Optional: Don't log access to assets
    }

    location ~ \.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    # Rule borrowed from `.htaccess`
    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
```
- Symlink configuration site available to site enabled

  `$ ln -s /etc/nginx/sites-available/nextcloud.conf /etc/nginx/sites-enabled/`

- Restart nginx and access on browser through server name
- Go through the installation of Nexcloud
- The user data on the installation dialog should point e.g to `administrator` or similar, that user will become adminitrative access rights in Nextcloud!

```
    Database user: nextcloud_user 
    Database password: The password you set for nextcloud_user (1234)
    Database name: nextcloud_db
    Database host: localhost (or your PostgreSQL server address)
```

# Optional other steps for more enhanced configuration modifications

## Sample Commands for File Permissions

- Move the data directory outside the web server document root and change it's permission

  `$ sudo mv /var/www/html/nextcloud/data /var/nextcloud_data`

  `$ sudo chown -R www-data:www-data /var/nextcloud_data`

  `$ chown -R www-data:www-data /var/www/nextcloud/`

- Update Nextcloud Configuration:

  1. Open the config/config.php file of your Nextcloud installation.

    `$ vi /var/www/nextcloud/config/config.php`

  2. Update the 'datadirectory' parameter to point to the new location of your data directory.

     ```  'datadirectory' => '/var/nextcloud_data' ```

- Restart Services:

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
