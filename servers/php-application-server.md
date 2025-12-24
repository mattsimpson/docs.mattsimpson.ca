---
order: 10
icon: rocket
---
# PHP 8.5 Application Server

*Last Update & Tested: 2025-12-24*

This documentation can be used as a reference to create a basic Linux web application server running Apache and PHP 8.5. This includes several optional steps if you would like to include an additional Apache VirtualHost for a staging environment or would like Supervisor to run Laravel `artisan` for Queues or WebSockets.

!!!warning
The hostname referenced throughout this documentation will be `app.example.org` and `staging.app.example.org`, which must be replaced by your actual hostname.
!!!

+++ Enterprise Linux 9

!!!
This Enterprise Linux 9 documentation should work with compatible derivatives of Fedora, including RedHat Enterprise, Rocky Linux, and AlmaLinux. If you find any subtle differences, please report the inconsistency or create a pull request with the improvements.
!!!

This documentation assumes a few things, including:
- you have done an installation from the Minimal ISO image.
- that your server has internet connectivity and a hostname (e.g., `app.example.org`).
- that you have SSH access to the server.

### Update operating system and install packages

1.  SSH into the server and `sudo` to root:
    ```bash
    ssh user@app.example.org
    sudo -s
    ```

2.  Set the hostname of the server using the NetworkManager CLI (`nmcli`) program:
    ```bash
    nmcli general hostname app.example.org
    ```

3.  Install the Extra Packages for Enterprise Linux (EPEL) and Remi PHP repositories:
    ```bash
    dnf -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm \
    https://rpms.remirepo.net/enterprise/remi-release-9.rpm
    ```
 
4. Enable the CodeReady Builder (CRB) repository frequently used by EPEL:
    ```bash
    dnf config-manager --set-enabled crb
    ```

5.  Install then run `screen` (a good practice before updating/installing packages over a remote connection):
    ```bash
    dnf -y install screen && screen
    ```

6.  Update the operating system:
    ```bash
    dnf -y update
    ```

    If a kernel update happened, you might as well reboot now:
    ```bash
    reboot
    ```

7.  Enable the Remi PHP repository and activate the PHP 8.5 module:
    ```bash
    dnf config-manager --set-enabled remi
    dnf module -y enable php:remi-8.5
    ```

8.  Install some awesome packages including Apache, OpenSSL, PHP, Git, MariaDB Client, Supervisor, SELinux Tools, and several other useful programs:
    ```bash
    dnf -y install curl \
                    git \
                    httpd \
                    mariadb \
                    mod_ssl \
                    openssl \
                    php-bcmath \
                    php-cli \
                    php-devel \
                    php-fpm \
                    php-gd \
                    php-imap \
                    php-intl \
                    php-json \
                    php-ldap \
                    php-mbstring \
                    php-mysqlnd \
                    php-pdo \
                    php-pecl-redis \
                    php-pecl-zip \
                    php-sodium \
                    php-tidy \
                    php-xml \
                    php-xmlrpc \
                    policycoreutils-python-utils \
                    supervisor \
                    unzip \
                    vim \
                    wget
    ```

9. If you have SELinux enabled and your PHP application will need to make connections to external servers (e.g., APIs, database servers), you will need to set the `httpd_can_network_connect` and `httpd_can_network_connect_db` options so that SELinux will allow Apache to make network and database connections:
    ```bash
    setsebool -P httpd_can_network_connect 1
    setsebool -P httpd_can_network_connect_db 1
    ```

   If you get an error there, you may need to reinstall the policy packages:
    ```bash
    dnf install -y --refresh selinux-policy-targeted
    ```

10. Start PHP-FPM, Apache, and Supervisor and set them to start when the system boots:
    ```bash
    systemctl enable --now php-fpm
    systemctl enable --now httpd
    systemctl enable --now supervisord
    ```

11. If you have firewalld running, allow HTTP and HTTPS traffic and reload the firewall:
    ```bash
    firewall-cmd --add-service=http --add-service=https --permanent
    firewall-cmd --reload
    ```
    
    If you don't have it running, you may wish to install and enable it:
    ```bash
    dnf install -y firewalld
    systemctl enable --now firewalld
    ```

### Create a production environment

1. Create a new service account called `production`, which will be used to deploy your PHP application with a tool like [Deployer](https://deployer.org):
    ```bash
    useradd -m production
    ```

2. Create and set permissions on the SSH `authorized_keys` file for the `production` user.
    ```bash
    mkdir /home/production/.ssh
    touch /home/production/.ssh/authorized_keys
    chown -R production:production /home/production/.ssh
    chmod 700 /home/production/.ssh
    chmod 600 /home/production/.ssh/authorized_keys
    ```

3. Add your SSH public key (the output of `cat ~/.ssh/id_rsa.pub` on your computer) to the new `authorized_keys` file.
    ```bash
    vim /home/production/.ssh/authorized_keys
    ```

4. Create and appropriately permission the Apache document root and application storage directories for production.
    ```bash
    mkdir -p /var/www/vhosts/app.example.org/storage/logs
    mkdir /var/www/vhosts/app.example.org/storage/cache
    chown -R production:production /var/www/vhosts/app.example.org
    chown production:apache /var/www/vhosts/app.example.org
    chmod 750 /var/www/vhosts/app.example.org
    ```

5. Configure SELinux so the PHP application can access and write to the storage directory, then apply the SELinux context changes:
    ```bash
    semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/vhosts/app.example.org/storage(/.*)?"
    semanage fcontext -a -t httpd_log_t "/var/www/vhosts/app.example.org/storage/logs(/.*)?"
    semanage fcontext -a -t httpd_cache_t "/var/www/vhosts/app.example.org/storage/cache(/.*)?"
    restorecon -R -v /var/www/vhosts/app.example.org
    ```

    !!!warning
    If your PHP application writes data to areas other than the previously defined `storage` directory, you will need to set and apply the `httpd_sys_rw_content_t` context to that location/file as well. For example, if you install Wordpress into the `/var/www/vhosts/app.example.org/current/public` directory, you would need to run:
    ```bash
    semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/vhosts/app.example.org/current/public(/.*)?"
    restorecon -R -v /var/www/vhosts/app.example.org
    ```
    !!!

6. Append `127.0.0.1    app.example.org` to the `/etc/hosts` file:
   ```bash
   echo "127.0.0.1    app.example.org" | tee -a /etc/hosts
   ```

### Create a staging environment

!!!
Setting up a staging VirtualHost is entirely optional. In most cases it is not advisable to run your staging environment on the same server as you production environment; however, it is technically possible and this documentation can be used as a reference regardless of whether staging is on the same environment as production or a dedicated server.
!!!

1. Create a new service account called `staging`, which will be used to deploy the staging version of your PHP application with a tool like [Deployer](https://deployer.org):
    ```bash
    useradd -m staging
    ```

2. Create and permission the SSH `authorized_keys` file for the `staging` user.
    ```bash
    mkdir /home/staging/.ssh
    touch /home/staging/.ssh/authorized_keys
    chown -R staging:staging /home/staging/.ssh
    chmod 700 /home/staging/.ssh
    chmod 600 /home/staging/.ssh/authorized_keys
    ```

3. Add your SSH public key to the new `authorized_keys` file.
    ```bash
    vim /home/staging/.ssh/authorized_keys
    ```

4. Create and appropriately permission the Apache document root and application storage directories for staging.
    ```bash
    mkdir -p /var/www/vhosts/staging.app.example.org/storage/logs
    mkdir /var/www/vhosts/staging.app.example.org/storage/cache
    chown -R staging:staging /var/www/vhosts/staging.app.example.org
    chown staging:apache /var/www/vhosts/staging.app.example.org
    chmod 750 /var/www/vhosts/staging.app.example.org
    ```

5. Configure SELinux so the PHP application can access and write to the storage directory, then apply the SELinux context changes:
    ```bash
    semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/vhosts/staging.app.example.org/storage(/.*)?"
    semanage fcontext -a -t httpd_log_t "/var/www/vhosts/staging.app.example.org/storage/logs(/.*)?"
    semanage fcontext -a -t httpd_cache_t "/var/www/vhosts/staging.app.example.org/storage/cache(/.*)?"
    restorecon -R -v /var/www/vhosts/staging.app.example.org
    ```

6. Append `127.0.0.1    staging.app.example.org` to the `/etc/hosts` file:
   ```bash
   echo "127.0.0.1    staging.app.example.org" | tee -a /etc/hosts
   ```

### Generate TLS certificates

1. Generate the private keys required for each of your hostnames:
    ```bash
    mkdir -p /root/certs
    openssl genrsa -out /root/certs/app.example.org.key 4096
    openssl genrsa -out /root/certs/staging.app.example.org.key 4096
    ```

2. Generate the TLS Certificate Signing Requests (CSR) that you will need to provide to your Certificate Authority when requesting a TLS (SSL) Certificate.

    Do this for `app.example.org`:
    ```bash
    openssl req -new -key /root/certs/app.example.org.key -out /root/certs/app.example.org.csr
    ```

    Do this for `staging.app.example.org`:
    ```bash
    openssl req -new -key /root/certs/staging.app.example.org.key -out /root/certs/staging.app.example.org.csr
    ```

    !!!
    You will be asked a number of questions, answer accordingly, but **do not answer** anything for "Email Address", "A challenge password", or "An optional company name."
    !!!

    ```
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [XX]:CA
    State or Province Name (full name) []:Ontario
    Locality Name (eg, city) [Default City]:Kingston
    Organization Name (eg, company) [Default Company Ltd]:Silentweb
    Organizational Unit Name (eg, section) []:Technical Documentation
    Common Name (eg, your name or your server's hostname) []:app.example.org
    Email Address []:

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    ```

3. Once you've received your new certificate from your Certificate Authority, create a .crt file for each hostname and paste in the contents of the certificate. You will also likely receive a root chain certificate, so paste the contents of that into a new file called `ca-certificate.crt` in the same directory.
    ```bash
    vim /root/certs/app.example.org.crt
    vim /root/certs/staging.app.example.org.crt
    vim /root/certs/ca-certificate.crt
    ```

4. If you are only creating self-signed certificates, you should do this for each hostname:
    ```bash
    openssl x509 -req -days 365 -in /root/certs/app.example.org.csr -signkey /root/certs/app.example.org.key -out /root/certs/app.example.org.crt
    openssl x509 -req -days 365 -in /root/certs/staging.app.example.org.csr -signkey /root/certs/staging.app.example.org.key -out /root/certs/staging.app.example.org.crt
    ```

5. Once finished, you will end up with a `/root/certs` directory that looks as follows:
    ```bash
    [root@app certs]# ls -1
    app.example.org.crt
    app.example.org.csr
    app.example.org.key
    ca-certificate.crt
    staging.app.example.org.crt
    staging.app.example.org.csr
    staging.app.example.org.key
    ```

6. Copy the certificates in the Apache VirtualHost directory:
    ```bash
    mkdir /var/www/vhosts/app.example.org/certs/
    cp /root/certs/app.example.org.crt /var/www/vhosts/app.example.org/certs/
    cp /root/certs/app.example.org.key /var/www/vhosts/app.example.org/certs/
    mkdir /var/www/vhosts/staging.app.example.org/certs/
    cp /root/certs/staging.app.example.org.crt /var/www/vhosts/staging.app.example.org/certs/
    cp /root/certs/staging.app.example.org.key /var/www/vhosts/staging.app.example.org/certs/
    ```

### Configure Apache and PHP

1. Create a new file called `/etc/php.d/app.ini` and add the following, making sure to adjust to your own [supported timezone](https://www.php.net/manual/en/timezones.php):
    ```
    date.timezone = America/Toronto
    display_errors = Off
    error_reporting = E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT
    expose_php = Off
    memory_limit = 512M
    post_max_size = 512M
    session.cookie_secure = 1
    session.cookie_httponly = 1
    session.cookie_samesite = Strict
    upload_max_filesize = 512M
    ```

2. Create a PHP-FPM Pool for the `production` user by creating a new file called `/etc/php-fpm.d/production.conf` and add the following:
    ```
    ; Start a new pool named 'production'.
    [production]

    ; Unix user/group of processes
    user = production
    group = production

    listen = /run/php-fpm/app.example.org.sock

    listen.acl_users = apache,nginx

    listen.allowed_clients = 127.0.0.1

    pm = dynamic
    pm.max_children = 50
    pm.start_servers = 5
    pm.min_spare_servers = 5
    pm.max_spare_servers = 35
    ```

3. Create a PHP-FPM Pool for the `staging` user by creating a new file called `/etc/php-fpm.d/staging.conf` and add the following:
    ```
    ; Start a new pool named 'staging'.
    [staging]

    ; Unix user/group of processes
    user = staging
    group = staging

    listen = /run/php-fpm/staging.app.example.org.sock

    listen.acl_users = apache,nginx

    listen.allowed_clients = 127.0.0.1

    pm = dynamic
    pm.max_children = 50
    pm.start_servers = 5
    pm.min_spare_servers = 5
    pm.max_spare_servers = 35
    ```

4. Create the Apache VirtualHost for `app.example.org` by creating a new file called `/etc/httpd/conf.d/000-app.example.org.conf` and add the following:
    ```
    # This will limit what information Apache reveals about itself.
    ServerTokens Prod
    ServerSignature Off
    TraceEnable Off

    # Enable OCSP stapling.
    SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

    # Production: app.example.org
    <VirtualHost *:80>
        ServerName app.example.org
        ServerAdmin hostmaster@example.org

        RewriteEngine On
        RewriteCond %{HTTPS} off
        RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    </VirtualHost>
    <VirtualHost *:443>
        ServerName app.example.org:443
        ServerAdmin hostmaster@example.org

        SSLEngine on
        SSLProtocol -all +TLSv1.2
        SSLHonorCipherOrder on
        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH

        SSLCertificateFile /var/www/vhosts/app.example.org/certs/app.example.org.crt
        SSLCertificateKeyFile /var/www/vhosts/app.example.org/certs/app.example.org.key
        #SSLCACertificateFile /var/www/vhosts/app.example.org/certs/ca-certificate.crt

        SSLUseStapling on

        Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains;"
        Header always set X-Frame-Options SAMEORIGIN

        DocumentRoot /var/www/vhosts/app.example.org/current/public
        <Directory "/var/www/vhosts/app.example.org/current/public">

            <IfModule mod_proxy_fcgi.c>
                <Files ~ (\.php$)>
                    SetHandler proxy:unix:/run/php-fpm/app.example.org.sock|fcgi://127.0.0.1:9000
                </Files>
            </IfModule>

            Options FollowSymLinks
            Require all granted
            AllowOverride all
        </Directory>

        # Using WebSockets? You can enable this by uncommenting the following lines.
        #ProxyPass "/app/" "ws://localhost:6000/app/"
        #ProxyPassReverse "/app/" "ws://localhost:6000/app/"
        #ProxyPass "/apps/" "http://localhost:6000/apps/"
        #ProxyPassReverse "/apps/" "http://localhost:6000/apps/"
    </VirtualHost>
    ```

5. Create the Apache VirtualHost for `staging.app.example.org` by creating a new file called `/etc/httpd/conf.d/000-staging.app.example.org.conf` and add the following:
    ```
    # This will limit what information Apache reveals about itself.
    ServerTokens Prod
    ServerSignature Off
    TraceEnable Off

    # Enable OCSP stapling.
    SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

    # Staging: staging.app.example.org
    <VirtualHost *:80>
        ServerName staging.app.example.org
        ServerAdmin hostmaster@example.org

        RewriteEngine On
        RewriteCond %{HTTPS} off
        RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    </VirtualHost>
    <VirtualHost *:443>
        ServerName staging.app.example.org:443
        ServerAdmin hostmaster@example.org

        SSLEngine on
        SSLProtocol -all +TLSv1.2
        SSLHonorCipherOrder on
        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH

        SSLCertificateFile /var/www/vhosts/staging.app.example.org/certs/staging.app.example.org.crt
        SSLCertificateKeyFile /var/www/vhosts/staging.app.example.org/certs/staging.app.example.org.key
        #SSLCACertificateFile /var/www/vhosts/staging.app.example.org/certs/ca-certificate.crt

        SSLUseStapling on

        Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains;"
        Header always set X-Frame-Options SAMEORIGIN

        DocumentRoot /var/www/vhosts/staging.app.example.org/current/public
        <Directory "/var/www/vhosts/staging.app.example.org/current/public">
            <IfModule mod_proxy_fcgi.c>
                <Files ~ (\.php$)>
                    SetHandler proxy:unix:/run/php-fpm/staging.app.example.org.sock|fcgi://127.0.0.1:9000
                </Files>
            </IfModule>

            Options FollowSymLinks
            Require all granted
            AllowOverride all
        </Directory>

        # Using WebSockets? You can enable this by uncommenting the following lines.
        #ProxyPass "/app/" "ws://localhost:6001/app/"
        #ProxyPassReverse "/app/" "ws://localhost:6001/app/"
        #ProxyPass "/apps/" "http://localhost:6001/apps/"
        #ProxyPassReverse "/apps/" "http://localhost:6001/apps/"
    </VirtualHost>
    ```

6. Test that your Apache configuration is okay, then restart Apache and PHP-FPM:
    ```bash
    apachectl configtest
    systemctl restart httpd
    systemctl restart php-fpm
    ```

### Configure Supervisor

Supervisor has many uses, but in this example we are using it to run Laravel Queue and Worker. If you don't need something like this running, simply skip this or use it as a reference later on.

1. Create a new file called `/etc/supervisord.d/app.example.org.ini` and use the following template snippet as a reference to create your own file.
    !!!
    Make sure you have the correct path in `command` and `stdout_logfile`, and that `user` matches the correct service account used to deploy your app.
    !!!

    ```
    [program:app-queue-production]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/app.example.org/current/artisan queue:work --queue=high,emails,default,low --env=production
    autostart=true
    autorestart=true
    user=production
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/app.example.org/storage/logs/worker.log

    [program:app-websocket-production]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/app.example.org/current/artisan websockets:serve --port=6000 --env=production
    autostart=true
    autorestart=true
    user=production
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/app.example.org/storage/logs/websocket.log

    [group:app-server]
    programs=app-queue-production,app-websocket-production
    ```

2. Create a new file called `/etc/supervisord.d/staging.app.example.org.ini` and use the following template snippet as a reference to create your own file.
    !!!
    Make sure you have the correct path in `command` and `stdout_logfile`, and that `user` matches the correct service account used to deploy your app.
    !!!

    ```
    [program:app-queue-staging]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/staging.app.example.org/current/artisan queue:work --queue=high,emails,default,low --env=staging
    autostart=true
    autorestart=true
    user=staging
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/staging.app.example.org/storage/logs/worker.log

    [program:app-websocket-staging]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/staging.app.example.org/current/artisan websockets:serve --port=6001 --env=staging
    autostart=true
    autorestart=true
    user=staging
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/staging.app.example.org/storage/logs/websocket.log

    [group:app-server]
    programs=app-queue-staging,app-websocket-staging
    ```

3. Restart Supervisor:
    ```
    systemctl restart supervisord
    ```

+++ Ubuntu 24.04

This documentation assumes a few things, including:
- you have done a fresh installation from Ubuntu ISO image.
- that your server has internet connectivity and a hostname (e.g., `app.example.org`).
- that you have SSH access to the server.

### Update operating system and install packages

1.  SSH into the server and `sudo` to root:
    ```bash
    ssh user@app.example.org
    sudo -s
    ```

2.  Set the hostname of the server using the `hostnamectl` program:
    ```bash
    hostnamectl set-hostname app.example.org
    ```

3.  Run `screen` (a good practice before updating/installing packages over a remote connection):
    ```bash
    screen
    ```

4.  Update the operating system:
    ```bash
    apt update
    apt upgrade
    ```

    If a kernel update happened, you might as well reboot now:
    ```bash
    reboot
    ```
    
5.  Install the package for the `add-apt-repository` tool and then add the `ondrej/php` Personal Package Archives (PPA).
    ```bash
    sudo apt install software-properties-common
    LC_ALL=C.UTF-8 sudo add-apt-repository ppa:ondrej/php
    ```
    !!!question No PPA Please
    Don't want to use a PPA? No problem. At this point you will just be limited to PHP 8.3. Just replace anywhere that says `php8.5` with `php8.3`.
    !!!

6.  Install some awesome packages including Apache, OpenSSL, PHP, Git, MariaDB Client, Supervisor, and several other useful programs:
    ```bash
    apt install -y apache2 \
                            curl \
                            git \
                            mariadb-client \
                            openssl \
                            php8.5-bcmath \
                            php8.5-cli \
                            php8.5-curl \
                            php8.5-fpm \
                            php8.5-gd \
                            php8.5-imap \
                            php8.5-intl \
                            php8.5-ldap \
                            php8.5-mbstring \
                            php8.5-mysql \
                            php8.5-redis \
                            php8.5-tidy \
                            php8.5-xml \
                            php8.5-xmlrpc \
                            php8.5-zip \
                            supervisor \
                            unzip \
                            wget
    ```

7.  Start PHP-FPM, Apache, and Supervisor and set them to start when the system boots:
    ```bash
    systemctl enable --now php8.5-fpm
    systemctl enable --now apache2
    systemctl enable --now supervisor
    ```

### Create a production environment

1. Create a new service account called `production`, which will be used to deploy your PHP application with a tool like [Deployer](https://deployer.org):
    ```bash
    useradd -m production
    ```

2. Create and set permissions on the SSH `authorized_keys` file for the `production` user.
    ```bash
    mkdir /home/production/.ssh
    touch /home/production/.ssh/authorized_keys
    chown -R production:production /home/production/.ssh
    chmod 700 /home/production/.ssh
    chmod 600 /home/production/.ssh/authorized_keys
    ```

3. Add your SSH public key (the output of `cat ~/.ssh/id_rsa.pub` on your computer) to the new `authorized_keys` file.
    ```bash
    vim /home/production/.ssh/authorized_keys
    ```

4. Create and appropriately permission the Apache document root and application storage directories for production.
    ```bash
    mkdir -p /var/www/vhosts/app.example.org/storage/logs
    mkdir /var/www/vhosts/app.example.org/storage/cache
    chown -R production:production /var/www/vhosts/app.example.org
    chown production:www-data /var/www/vhosts/app.example.org
    chmod 750 /var/www/vhosts/app.example.org
    ```

5. Append `127.0.0.1    app.example.org` to the `/etc/hosts` file:
   ```bash
   echo "127.0.0.1    app.example.org" | tee -a /etc/hosts
   ```

### Create a staging environment

!!!
Setting up a staging VirtualHost is entirely optional. In most cases it is not advisable to run your staging environment on the same server as you production environment; however, it is technically possible and this documentation can be used as a reference regardless of whether staging is on the same environment as production or a dedicated server.
!!!

1. Create a new service account called `staging`, which will be used to deploy the staging version of your PHP application with a tool like [Deployer](https://deployer.org):
    ```bash
    useradd -m staging
    ```

2. Create and permission the SSH `authorized_keys` file for the `staging` user.
    ```bash
    mkdir /home/staging/.ssh
    touch /home/staging/.ssh/authorized_keys
    chown -R staging:staging /home/staging/.ssh
    chmod 700 /home/staging/.ssh
    chmod 600 /home/staging/.ssh/authorized_keys
    ```

3. Add your SSH public key to the new `authorized_keys` file.
    ```bash
    vim /home/staging/.ssh/authorized_keys
    ```

4. Create and appropriately permission the Apache document root and application storage directories for staging.
    ```bash
    mkdir -p /var/www/vhosts/staging.app.example.org/storage/logs
    mkdir /var/www/vhosts/staging.app.example.org/storage/cache
    chown -R staging:staging /var/www/vhosts/staging.app.example.org
    chown staging:www-data /var/www/vhosts/staging.app.example.org
    chmod 750 /var/www/vhosts/staging.app.example.org
    ```

5. Append `127.0.0.1    staging.app.example.org` to the `/etc/hosts` file:
   ```bash
   echo "127.0.0.1    staging.app.example.org" | tee -a /etc/hosts
   ```

### Generate TLS certificates

1. Generate the private keys required for each of your hostnames:
    ```bash
    mkdir -p /root/certs
    openssl genrsa -out /root/certs/app.example.org.key 4096
    openssl genrsa -out /root/certs/staging.app.example.org.key 4096
    ```

2. Generate the TLS Certificate Signing Requests (CSR) that you will need to provide to your Certificate Authority when requesting a TLS (SSL) Certificate.

    Do this for `app.example.org`:
    ```bash
    openssl req -new -key /root/certs/app.example.org.key -out /root/certs/app.example.org.csr
    ```

    Do this for `staging.app.example.org`:
    ```bash
    openssl req -new -key /root/certs/staging.app.example.org.key -out /root/certs/staging.app.example.org.csr
    ```

    !!!
    You will be asked a number of questions, answer accordingly, but **do not answer** enter anything for "Email Address", "A challenge password", or "An optional company name."
    !!!

    ```
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [XX]:CA
    State or Province Name (full name) []:Ontario
    Locality Name (eg, city) [Default City]:Kingston
    Organization Name (eg, company) [Default Company Ltd]:Silentweb
    Organizational Unit Name (eg, section) []:Technical Documentation
    Common Name (eg, your name or your server's hostname) []:app.example.org
    Email Address []:

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    ```

3. Once you've received your new certificate from your Certificate Authority, create a .crt file for each hostname and paste in the contents of the certificate. You will also likely receive a root chain certificate, so paste the contents of that into a new file called `ca-certificate.crt` in the same directory.
    ```bash
    vim /root/certs/app.example.org.crt
    vim /root/certs/staging.app.example.org.crt
    vim /root/certs/ca-certificate.crt
    ```

4. If you are only creating self-signed certificates, you should do this for each hostname:
    ```bash
    openssl x509 -req -days 365 -in /root/certs/app.example.org.csr -signkey /root/certs/app.example.org.key -out /root/certs/app.example.org.crt
    openssl x509 -req -days 365 -in /root/certs/staging.app.example.org.csr -signkey /root/certs/staging.app.example.org.key -out /root/certs/staging.app.example.org.crt
    ```

5. Once finished, you will end up with a `/root/certs` directory that looks as follows:
    ```bash
    [root@app certs]# ls -1
    app.example.org.crt
    app.example.org.csr
    app.example.org.key
    ca-certificate.crt
    staging.app.example.org.crt
    staging.app.example.org.csr
    staging.app.example.org.key
    ```

6. Copy the certificates in the Apache VirtualHost directory:
    ```bash
    mkdir /var/www/vhosts/app.example.org/certs/
    cp /root/certs/app.example.org.crt /var/www/vhosts/app.example.org/certs/
    cp /root/certs/app.example.org.key /var/www/vhosts/app.example.org/certs/
    mkdir /var/www/vhosts/staging.app.example.org/certs/
    cp /root/certs/staging.app.example.org.crt /var/www/vhosts/staging.app.example.org/certs/
    cp /root/certs/staging.app.example.org.key /var/www/vhosts/staging.app.example.org/certs/
    ```

### Configure Apache and PHP

1. Enable several associated Apache modules and the Apache PHP configuration:
    ```bash
    a2enmod ssl rewrite headers proxy proxy_http proxy_balancer expires proxy_fcgi setenvif
    a2enconf php8.5-fpm
    ```

2. Create a new file called `/etc/php/8.5/mods-available/app.ini` and add the following, making sure to adjust to your own [supported timezone](https://www.php.net/manual/en/timezones.php):
    ```
    date.timezone = America/Toronto
    display_errors = Off
    error_reporting = E_ALL & ~E_NOTICE & ~E_DEPRECATED & ~E_STRICT
    expose_php = Off
    memory_limit = 512M
    post_max_size = 512M
    session.cookie_secure = 1
    session.cookie_httponly = 1
    session.cookie_samesite = Strict
    upload_max_filesize = 512M
    ```

3. Enable these new PHP settings by typing:
    ```bash
    phpenmod app
    ```

4. Create a PHP-FPM Pool for the `production` user by creating a new file called `/etc/php/8.5/fpm/pool.d/production.conf` and add the following:
    ```
    ; Start a new pool named 'production'.
    [production]

    ; Unix user/group of processes
    user = production
    group = www-data

    listen = /run/php/app.example.org.sock
    listen.owner = production
    listen.group = www-data
    listen.allowed_clients = 127.0.0.1

    pm = dynamic
    pm.max_children = 50
    pm.start_servers = 5
    pm.min_spare_servers = 5
    pm.max_spare_servers = 35
    ```

5. Create a PHP-FPM Pool for the `staging` user by creating a new file called `/etc/php/8.5/fpm/pool.d/staging.conf` and add the following:
    ```
    ; Start a new pool named 'staging'.
    [staging]

    ; Unix user/group of processes
    user = staging
    group = staging

    listen = /run/php/staging.app.example.org.sock
    listen.owner = staging
    listen.group = www-data
    listen.allowed_clients = 127.0.0.1

    pm = dynamic
    pm.max_children = 50
    pm.start_servers = 5
    pm.min_spare_servers = 5
    pm.max_spare_servers = 35
    ```

6. Create the Apache VirtualHost for `app.example.org` by creating a new file called `/etc/apache2/sites-available/000-app.example.org.conf` and add the following:
    ```
    # This will limit what information Apache reveals about itself.
    ServerTokens Prod
    ServerSignature Off
    TraceEnable Off

    # Enable OCSP stapling.
    SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

    # Production: app.example.org
    <VirtualHost *:80>
        ServerName app.example.org
        ServerAdmin hostmaster@example.org

        RewriteEngine On
        RewriteCond %{HTTPS} off
        RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    </VirtualHost>
    <VirtualHost *:443>
        ServerName app.example.org:443
        ServerAdmin hostmaster@example.org

        SSLEngine on
        SSLProtocol -all +TLSv1.2
        SSLHonorCipherOrder on
        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH

        SSLCertificateFile /var/www/vhosts/app.example.org/certs/app.example.org.crt
        SSLCertificateKeyFile /var/www/vhosts/app.example.org/certs/app.example.org.key
        #SSLCACertificateFile /var/www/vhosts/app.example.org/certs/ca-certificate.crt

        SSLUseStapling on

        Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains;"
        Header always set X-Frame-Options SAMEORIGIN

        DocumentRoot /var/www/vhosts/app.example.org/current/public
        <Directory "/var/www/vhosts/app.example.org/current/public">

            <IfModule mod_proxy_fcgi.c>
                <Files ~ (\.php$)>
                    SetHandler proxy:unix:/run/php/app.example.org.sock|fcgi://127.0.0.1:9000
                </Files>
            </IfModule>

            Options FollowSymLinks
            Require all granted
            AllowOverride all
        </Directory>

        # Using WebSockets? You can enable this by uncommenting the following lines.
        #ProxyPass "/app/" "ws://localhost:6000/app/"
        #ProxyPassReverse "/app/" "ws://localhost:6000/app/"
        #ProxyPass "/apps/" "http://localhost:6000/apps/"
        #ProxyPassReverse "/apps/" "http://localhost:6000/apps/"
    </VirtualHost>
    ```

7. Create the Apache VirtualHost for `staging.app.example.org` by creating a new file called `/etc/apache2/sites-available/000-staging.app.example.org.conf` and add the following:
    ```
    # This will limit what information Apache reveals about itself.
    ServerTokens Prod
    ServerSignature Off
    TraceEnable Off

    # Enable OCSP stapling.
    SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

    # Staging: staging.app.example.org
    <VirtualHost *:80>
        ServerName staging.app.example.org
        ServerAdmin hostmaster@example.org

        RewriteEngine On
        RewriteCond %{HTTPS} off
        RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
    </VirtualHost>
    <VirtualHost *:443>
        ServerName staging.app.example.org:443
        ServerAdmin hostmaster@example.org

        SSLEngine on
        SSLProtocol -all +TLSv1.2
        SSLHonorCipherOrder on
        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH

        SSLCertificateFile /var/www/vhosts/staging.app.example.org/certs/staging.app.example.org.crt
        SSLCertificateKeyFile /var/www/vhosts/staging.app.example.org/certs/staging.app.example.org.key
        #SSLCACertificateFile /var/www/vhosts/staging.app.example.org/certs/ca-certificate.crt

        SSLUseStapling on

        Header always set Strict-Transport-Security "max-age=63072000; includeSubdomains;"
        Header always set X-Frame-Options SAMEORIGIN

        DocumentRoot /var/www/vhosts/staging.app.example.org/current/public
        <Directory "/var/www/vhosts/staging.app.example.org/current/public">
            <IfModule mod_proxy_fcgi.c>
                <Files ~ (\.php$)>
                    SetHandler proxy:unix:/run/php/staging.app.example.org.sock|fcgi://127.0.0.1:9000
                </Files>
            </IfModule>

            Options FollowSymLinks
            Require all granted
            AllowOverride all
        </Directory>

        # Using WebSockets? You can enable this by uncommenting the following lines.
        #ProxyPass "/app/" "ws://localhost:6001/app/"
        #ProxyPassReverse "/app/" "ws://localhost:6001/app/"
        #ProxyPass "/apps/" "http://localhost:6001/apps/"
        #ProxyPassReverse "/apps/" "http://localhost:6001/apps/"
    </VirtualHost>
    ```

8. Enable `app.example.org` and `staging.app.example.org`:
    ```
    a2ensite 000-app.example.org
    a2ensite 000-staging.app.example.org
    ```

9. Add the `production` and `staging` user to the `www-data` group.
   ```bash
   usermod -aG production www-data
   usermod -aG staging www-data
   ```

10. Test that your Apache configuration is okay, then restart Apache and PHP-FPM:
    ```bash
    apachectl configtest
    systemctl restart apache2
    systemctl restart php8.5-fpm
    ```

### Configure Supervisor

Supervisor has many uses, but in this example we are using it to run Laravel Queue and Worker. If you don't need something like this running, simply skip this or use it as a reference later on.

1. Create a new file called `/etc/supervisor/conf.d/app.example.org.ini` and use the following template snippet as a reference to create your own file.
    !!!
    Make sure you have the correct path in `command` and `stdout_logfile`, and that `user` matches the correct service account used to deploy your app.
    !!!

    ```
    [program:app-queue-production]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/app.example.org/current/artisan queue:work --queue=high,emails,default,low --env=production
    autostart=true
    autorestart=true
    user=production
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/app.example.org/storage/logs/worker.log

    [program:app-websocket-production]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/app.example.org/current/artisan websockets:serve --port=6000 --env=production
    autostart=true
    autorestart=true
    user=production
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/app.example.org/storage/logs/websocket.log

    [group:app-server]
    programs=app-queue-production,app-websocket-production
    ```

2. Create a new file called `/etc/supervisor/conf.d/staging.app.example.org.ini` and use the following template snippet as a reference to create your own file.
    !!!
    Make sure you have the correct path in `command` and `stdout_logfile`, and that `user` matches the correct service account used to deploy your app.
    !!!

    ```
    [program:app-queue-staging]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/staging.app.example.org/current/artisan queue:work --queue=high,emails,default,low --env=staging
    autostart=true
    autorestart=true
    user=staging
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/staging.app.example.org/storage/logs/worker.log

    [program:app-websocket-staging]
    process_name=%(program_name)s_%(process_num)02d
    command=php /var/www/vhosts/staging.app.example.org/current/artisan websockets:serve --port=6001 --env=staging
    autostart=true
    autorestart=true
    user=staging
    numprocs=1
    redirect_stderr=true
    stdout_logfile=/var/www/vhosts/staging.app.example.org/storage/logs/websocket.log

    [group:app-server]
    programs=app-queue-staging,app-websocket-staging
    ```

3. Restart Supervisor:
    ```
    systemctl restart supervisor
    ```

+++
