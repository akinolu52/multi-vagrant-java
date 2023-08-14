## Steps to configure and setup multi-vagrant for deploying a JAVA project (with Database, MemCache, RabbitMQ, Tomcat, Ngnix)

### Stack involved are

- Ngnix: Web service
- Tomcat: Application server
- RabbitMQ: Broker/Queuing Agent
- Memcache: DB Caching
- ElasticSearch: Indexing/Search service (RAM intensive)
- MySQL: SQL Database

### Steps involved includes:

1. Set up your multi-vagrant file for the various service then start the multi-vagrant

    ```bash
    vagrant up
    ```

    1.1 The hostmanager plugin (extension) is used configure the hostname and the matching IP address to all the vms provisioned. (NB: this plugin should be used globally)

    ```bash
     config.hostmanager.enabled = true 
     config.hostmanager.manage_host = true
    ```

    to verify the hosts entries use the command after you login `cat /etc/hosts`; the hostmanager makes sure that all the vms has an entry of all the defined hostname and its IP address

2. test the connection between the vms using the following commands:

    ```bash
    vagrant ssh app01

    ping db01 -c 4

    ping web01 -c 4
    
    ping mc01 -c 4

    ping rmq01 -c 4
    ```

    ideally you should get success response. (NB we limit the packet test to four using `-c 4` flag)

3. Set up the MySQL database

    3.1 login into the database vms

    ```bash
    vagrant up db01
    ```

    3.2 switch to root user

    ```bash
    sudo -i
    ```

    3.3 update yum and accept using `-y` flag

    ```bash
    yum update -y
    ```

    3.4 install epel-release (this gives access to more packages)

    ```bash
    yum install epel-release -y
    ```

    > [!IMPORTANT]
    > For m1 arch use the following command: `sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm`

    3.5 install git (for cloning the source repository) and maria-db (for the latest release of the database)

    ```bash
    yum install git mariadb-server -y
    ```

    3.6 start, enable and check the status of the mariadb server

    ```bash
    systemctl start mariadb

    systemctl enable mariadb

    systemctl status mariadb
    systemctl is-enabled mariadb
    ```

    3.7 run the secure installation script and follow the process

    ```bash
    mysql_secure_installation
    ```

    3.8 setup database and user

    ```SQL
    mysql -u root -p<your-password>

    create database accounts;

    show databases;

    grant all privileges on accounts.* TO 'admin'@'%' identified by '<your-password>';

    FLUSH PRIVILEGES;

    exit;
    ```

    3.9 Restart mariadb service

    ```bash
    systemctl restart mariadb
    ```

    3.10 clone the code and init your db

    ```bash
    git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git

    cd vprofile-project

    mysql -u root -p<your-password> accounts < src/main/resources/db_backup.sql

    mysql -u root -p<your-password> accounts
    ```

    ```SQL
    show tables;

    exit;
    ```

    ```bash
    systemctl restart mariadb
    ```

    nb the `-b` flag to git clone, is used to pick a specific branch

4. Setup up the Memcache service
