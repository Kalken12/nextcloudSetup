# nextcloudSetup

# Nextcloud installation 
#### Step 1: Install PHP and PHP extensions for Nextcloud
`$ sudo apt install php php-cli php-fpm php-json php-common php-mysql php-zip php-gd php-intl php-curl php-xml php-mbstring php-bcmath php-gmp`

#### check php version
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
