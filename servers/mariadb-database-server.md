---
order: 9
icon: database
---
# MariaDB Database Server

This documentation can be used as a reference to create a MariaDB database server. This includes an optional step if you are also looking to set them up in a Primary / Replica manual failover configuration.

!!!
The hostnames and IP addresses referenced throughout this documentation will be `app.example.org:192.168.25.5`, `db01.example.org:192.168.25.10`, and `db02.example.org:192.168.25.11`. These entries must be replaced by your actual hostnames and IP addresses.
!!!

### Primary MariaDB Server

+++ Enterprise Linux 9

!!!
This Enterprise Linux 9 documentation should work with compatible derivatives of Fedora, including RedHat Enterprise, Rocky Linux, and AlmaLinux. If you find any subtle differences, please report the inconsistency or create a pull request with the improvements.
!!!

This documentation assumes a few things, including:
- you have done an installation from the Minimal ISO image.
- that your server has internet connectivity and a hostname (e.g., `db01.example.org`).
- that you have SSH access to the server.

### Update operating system and install packages

1.  SSH into the server and `sudo` to root:
    ```bash
    ssh user@db01.example.org
    sudo -s
    ```

2.  Set the hostname of the server using the NetworkManager CLI (`nmcli`) program:
    ```bash
    nmcli general hostname db01.example.org
    ```

3.  Enable the CodeReady Builder (CRB) repository frequently used by EPEL:
    ```bash
    dnf config-manager --set-enabled crb
    ```

4.  Install the Extra Packages for Enterprise Linux (EPEL) repository:
    ```bash
    dnf -y install \
    https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
    ```

5.  Install `vim` and `screen`, then run `screen` (a good practice before updating/installing packages over a remote connection):
    ```bash
    dnf -y install vim screen && screen
    ```

6.  Update the operating system:
    ```bash
    dnf -y update
    ```

    If a kernel update happened, you might as well reboot now:
    ```bash
    reboot
    ```

7.  Install the NTP, MariaDB Client, and Server packages:
    ```bash
    dnf -y install chrony \
                    mariadb \
                    mariadb-server
    ```

8.  Create a new file called `/etc/my.cnf.d/performance.cnf` and place the following configuration within the file:
    !!!warning
    Do not forget to update the unique `server_id` variable if you're enabling server replication. The value needs to be between 1 and 4294967295. Pro tip: Set this to the IP address of the database server (e.g., 192.168.25.10 = 1921682510).
    !!!

    ```
    [mysqld]
    # Skip reverse DNS lookup
    skip-name-resolve = on                      # Recommended for performance, enabling this requires local connections to be 127.0.0.1 vs. localhost.

    # Innodb
    innodb_buffer_pool_size = 6G                # Main memory buffer of Innodb, very important.
    innodb_log_file_size = 1G                   # Recommended value from MySQLTuner.
    innodb_flush_method = O_DIRECT_NO_FSYNC     # Recommended for performance.
    innodb_flush_neighbors = 0                  # Recommended for performance.
    innodb_flush_log_at_trx_commit = 2          # Writes to OS, fsynced once per second.
    innodb_buffer_pool_load_at_startup = on
    innodb_buffer_pool_dump_at_shutdown = on
    innodb_ft_min_token_size = 3
    innodb_io_capacity = 450                    # Recommended for performance.
    innodb_random_read_ahead = on               # Recommended for performance.

    # Basic Settings
    thread_cache_size = 8
    table_open_cache = 4000
    table_definition_cache = 4617               # Recommended value from MySQLTuner.
    query_cache_size = 128M                     # Recommended value from MySQLTuner.
    query_cache_type = 1
    max_allowed_packet = 16777216
    join_buffer_size = 524288                   # Recommended value from MySQLTuner.
    tmp_table_size = 32M                        # Recommended value from MySQLTuner.
    max_heap_table_size = 32M                   # Recommended value from MySQLTuner.
    key_buffer_size = 26M                       # Recommended value from MySQLTuner.

    # Connections
    max_connections = 512                       # Optional: Increase max_connections from default of 151, if you have the resources.

    # Slow Query Logging / Tuning
    slow_query_log = on
    log_slow_verbosity = 'innodb,query_plan'
    long_query_time = 10
    performance_schema = on

    # Replication (OPTIONAL)
    #server_id = 1921682510                     # A unique ID for this server. Tip: Set to the IP address of the server.
    #log_bin = /var/lib/mysql/mysql-bin
    #expire_logs_days = 14
    #sync_binlog = 4                            # 1 = every transaction; 4 or 5 = every 4th or 5th transaction.
    ```

9.  Start MariaDB and set to start on system startup:
    ```bash
    systemctl daemon-reload
    systemctl enable --now mariadb
    ```

10. Run the `mysql_secure_installation` script included with MariaDB to tighten up the security of new installation:
    ```bash
    /usr/bin/mysql_secure_installation
    ```

    !!!
    By default there is no root password, so you should press enter for no password when prompted. Don't forget to set the root password during the "Setting the root password" step, regardless of whether it reports there being a password or not.
    !!!

    Here is the output of the `mysql_secure_installation` hardening:
    ```
    [root@db01]# /usr/bin/mysql_secure_installation

    NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
        SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

    In order to log into MariaDB to secure it, we'll need the current
    password for the root user. If you've just installed MariaDB, and
    haven't set the root password yet, you should just press enter here.

    Enter current password for root (enter for none): 
    OK, successfully used password, moving on...

    Setting the root password or using the unix_socket ensures that nobody
    can log into the MariaDB root user without the proper authorisation.

    You already have your root account protected, so you can safely answer 'n'.

    Switch to unix_socket authentication [Y/n] n
    ... skipping.

    You already have your root account protected, so you can safely answer 'n'.

    Change the root password? [Y/n] Y
    New password: 
    Re-enter new password: 
    Password updated successfully!
    Reloading privilege tables..
    ... Success!

    By default, a MariaDB installation has an anonymous user, allowing anyone
    to log into MariaDB without having to have a user account created for
    them.  This is intended only for testing, and to make the installation
    go a bit smoother.  You should remove them before moving into a
    production environment.

    Remove anonymous users? [Y/n] Y
    ... Success!

    Normally, root should only be allowed to connect from 'localhost'.  This
    ensures that someone cannot guess at the root password from the network.

    Disallow root login remotely? [Y/n] Y
    ... Success!

    By default, MariaDB comes with a database named 'test' that anyone can
    access.  This is also intended only for testing, and should be removed
    before moving into a production environment.

    Remove test database and access to it? [Y/n] Y
    - Dropping test database...
    ... Success!
    - Removing privileges on test database...
    ... Success!

    Reloading the privilege tables will ensure that all changes made so far
    will take effect immediately.

    Reload privilege tables now? [Y/n] Y
    ... Success!

    Cleaning up...

    All done!  If you've completed all of the above steps, your MariaDB
    installation should now be secure.

    Thanks for using MariaDB!
    ```

11. You can now connect to the database server as the `root` user:
    ```bash
    mysql -uroot
    ```

    If you would like to create a new database and associated user account, you can use the following queries as a template.

    !!!warning
    Don't forget to replace any instance of `wordpress` with the actual user and database name, and `192.168.25.5` with the IP address of your PHP Application Server.
    !!!

    ```sql
    CREATE DATABASE IF NOT EXISTS wordpress;
    CREATE USER 'wordpress'@'192.168.25.5' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'192.168.25.5';
    FLUSH PRIVILEGES;
    ```

12. Allow TCP traffic on port 3306 from your PHP Application Server and reload the firewall:
    ```bash
    firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.25.5" port port="3306" protocol="tcp" accept' --permanent
    firewall-cmd --reload
    ```

+++ Ubuntu 24.04

This documentation assumes a few things, including:
- you have done a fresh installation from Ubuntu ISO image.
- that your server has internet connectivity and a hostname (e.g., `db01.example.org`).
- that you have SSH access to the server.

### Update operating system and install packages

1.  SSH into the server and `sudo` to root:
    ```bash
    ssh user@app.example.org
    sudo -s
    ```

2.  Set the hostname of the server using the `hostnamectl` program:
    ```bash
    hostnamectl set-hostname db01.example.org
    ```

3.  Run `screen` (a good practice before updating/installing packages over a remote connection):
    ```bash
    screen
    ```

4.  Update the operating system:
    ```bash
    apt-get update
    apt-get upgrade
    ```

    If a kernel update happened, you might as well reboot now:
    ```bash
    reboot
    ```

5.  Install the MariaDB Client and Server packages:
    ```bash
    apt-get install install chrony \
                    mariadb-client \
                    mariadb-server
    ```

6.  Create a new file called `/etc/mysql/conf.d/performance.cnf` and place the following configuration within the file:
    !!!warning
    Do not forget to update the unique `server_id` variable if you're enabling server replication. The value needs to be between 1 and 4294967295. Pro tip: Set this to the IP address of the database server (e.g., 192.168.25.10 = 1921682510).
    !!!

    ```
    [mysqld]
    # Skip reverse DNS lookup
    skip-name-resolve = on                      # Recommended for performance, enabling this requires local connections to be 127.0.0.1 vs. localhost.

    # Innodb
    innodb_buffer_pool_size = 6G                # Main memory buffer of Innodb, very important.
    innodb_log_file_size = 1G                   # Recommended value from MySQLTuner.
    innodb_flush_method = O_DIRECT_NO_FSYNC     # Recommended for performance.
    innodb_flush_neighbors = 0                  # Recommended for performance.
    innodb_flush_log_at_trx_commit = 2          # Writes to OS, fsynced once per second.
    innodb_buffer_pool_load_at_startup = on
    innodb_buffer_pool_dump_at_shutdown = on
    innodb_ft_min_token_size = 3
    innodb_io_capacity = 450                    # Recommended for performance.
    innodb_random_read_ahead = on               # Recommended for performance.

    # Basic Settings
    thread_cache_size = 8
    table_open_cache = 4000
    table_definition_cache = 4617               # Recommended value from MySQLTuner.
    query_cache_size = 128M                     # Recommended value from MySQLTuner.
    query_cache_type = 1
    max_allowed_packet = 16777216
    join_buffer_size = 524288                   # Recommended value from MySQLTuner.
    tmp_table_size = 32M                        # Recommended value from MySQLTuner.
    max_heap_table_size = 32M                   # Recommended value from MySQLTuner.
    key_buffer_size = 26M                       # Recommended value from MySQLTuner.

    # Connections
    max_connections = 512                       # Optional: Increase max_connections from default of 151, if you have the resources.

    # Slow Query Logging / Tuning
    slow_query_log = on
    log_slow_verbosity = 'innodb,query_plan'
    long_query_time = 10
    performance_schema = on

    # Replication (OPTIONAL)
    #server_id = 1921682510                     # A unique ID for this server. Tip: Set to the IP address of the server.
    #log_bin = /var/lib/mysql/mysql-bin
    #expire_logs_days = 14
    #sync_binlog = 4                            # 1 = every transaction; 4 or 5 = every 4th or 5th transaction.
    ```

9.  Start MariaDB and set to start on system startup:
    ```bash
    systemctl daemon-reload
    systemctl enable --now mariadb
    ```

10. Run the `mysql_secure_installation` script included with MariaDB to tighten up the security of new installation:
    ```bash
    /usr/bin/mysql_secure_installation
    ```

    !!!
    By default there is no root password, so you should press enter for no password when prompted. Don't forget to set the root password during the "Setting the root password" step, regardless of whether it reports there being a password or not.
    !!!

    Here is the output of the `mysql_secure_installation` hardening:
    ```
    [root@db01]# /usr/bin/mysql_secure_installation

    NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
        SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

    In order to log into MariaDB to secure it, we'll need the current
    password for the root user. If you've just installed MariaDB, and
    haven't set the root password yet, you should just press enter here.

    Enter current password for root (enter for none): 
    OK, successfully used password, moving on...

    Setting the root password or using the unix_socket ensures that nobody
    can log into the MariaDB root user without the proper authorisation.

    You already have your root account protected, so you can safely answer 'n'.

    Switch to unix_socket authentication [Y/n] n
    ... skipping.

    You already have your root account protected, so you can safely answer 'n'.

    Change the root password? [Y/n] Y
    New password: 
    Re-enter new password: 
    Password updated successfully!
    Reloading privilege tables..
    ... Success!

    By default, a MariaDB installation has an anonymous user, allowing anyone
    to log into MariaDB without having to have a user account created for
    them.  This is intended only for testing, and to make the installation
    go a bit smoother.  You should remove them before moving into a
    production environment.

    Remove anonymous users? [Y/n] Y
    ... Success!

    Normally, root should only be allowed to connect from 'localhost'.  This
    ensures that someone cannot guess at the root password from the network.

    Disallow root login remotely? [Y/n] Y
    ... Success!

    By default, MariaDB comes with a database named 'test' that anyone can
    access.  This is also intended only for testing, and should be removed
    before moving into a production environment.

    Remove test database and access to it? [Y/n] Y
    - Dropping test database...
    ... Success!
    - Removing privileges on test database...
    ... Success!

    Reloading the privilege tables will ensure that all changes made so far
    will take effect immediately.

    Reload privilege tables now? [Y/n] Y
    ... Success!

    Cleaning up...

    All done!  If you've completed all of the above steps, your MariaDB
    installation should now be secure.

    Thanks for using MariaDB!
    ```

11. You can now connect to the database server as the `root` user:
    ```bash
    mysql -uroot
    ```

    If you would like to create a new database and associated user account, you can use the following queries as a template.

    !!!warning
    Don't forget to replace any instance of `wordpress` with the actual user and database name, and `192.168.25.5` with the IP address of your PHP Application Server.
    !!!

    ```sql
    CREATE DATABASE IF NOT EXISTS wordpress;
    CREATE USER 'wordpress'@'192.168.25.5' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpress'@'192.168.25.5';
    FLUSH PRIVILEGES;
    ```

+++

### Replica Database Server

If you intend on setting up a Replica Database Server, repeat each step outlined in the Primary Database Server section on your second `db02.example.org` server so that you have two freshly installed MariaDB database servers. Once you have finished doing that, you can proceed with the following changes.

#### Changes on db01.example.org

1. Connect with the root MariaDB user account on `db01.example.org`:
    ```bash
    mysql -uroot
    ```

2. Create a `repl` database user that will be able to connect from the `db02.example.org` server:
    !!!warning
    Don't forget to set a password by replacing `set-a-new-complex-password-here` with a complex password, and `192.168.25.11` with the IP address of your `db02.example.org` server.
    !!!

    ```sql
    GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.25.11' IDENTIFIED BY 'set-a-new-complex-password-here';
    ```

2. Run the follow query and make note of the master log file (i.e. `mysql-bin.000002`) as you will need it when configuring the replica:
    ```sql
    SHOW MASTER STATUS \G
    ```

#### Changes on db02.example.org

1. Connect with the root MariaDB user account on `db02.example.org`:
    ```bash
    mysql -uroot
    ```

2. Tell the replica server to replicate from your `db01.example.org` primary:
    !!!warning
    Don't forget to replace `set-a-new-complex-password-here` with the password you set previously, and `192.168.25.10` with the IP address of your `db01.example.org` server. Lastly, if the master log file you noted previously was not `mysql-bin.000002` then you will also need to update this.
    !!!

    ```
    CHANGE MASTER TO MASTER_HOST='192.168.25.10', MASTER_USER='repl', MASTER_PASSWORD='your-password-needs-to-go-here', MASTER_LOG_FILE='mysql-bin.000002';
    ```

3. Start the replication:
    ```sql
    START SLAVE;
    ```

    You can ensure that there are no errors and watch the status of the replication by running:
    ```sql
    SHOW MASTER STATUS \G
    ```
