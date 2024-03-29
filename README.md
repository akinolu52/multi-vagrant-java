## Steps to configure and setup multi-vagrant for deploying a JAVA project (with Database, MemCache, RabbitMQ, Tomcat, Nginx)

### Stack involved are

- Nginx: Web service
- Tomcat: Application server
- RabbitMQ: Broker/Queuing Agent
- Memcache: DB Caching
- ElasticSearch: Indexing/Search service (RAM intensive)
- MySQL: SQL Database

### Steps involved includes

1. Set up your multi-vagrant file for the various service then start the multi-vagrant

    ```bash
    vagrant up
    ```

    1.1 The hostmanager plugin (extension) is used configure the hostname and the matching IP address to all the vms provisioned. (NB: this plugin should be used globally)

    ```bash
    vagrant plugin install vagrant-hostmanager
    ```

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
    vagrant ssh db01
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
    > For m1 arch use the following command: `sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm -y`

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

4. Setup the Memcache service

    4.1 login into the Memcache vms

    ```bash
    vagrant ssh mc01
    ```

    4.2 switch to root user

    ```bash
    sudo -i
    ```

    4.3 update yum and accept using `-y` flag

    ```bash
    yum update -y
    ```

    4.4 install epel-release (this gives access to more packages)

    ```bash
    yum install epel-release -y
    ```

    > [!IMPORTANT]
    > For m1 arch use the following command: `sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm -y`

    4.5 install Memcache

    ```bash
    yum install memcached -y
    ```

    4.6 start, enable and check the status of the Memcache service

    ```bash
    systemctl start memcached

    systemctl enable memcached

    systemctl status memcached
    systemctl is-enabled memcached
    ```

    > [!IMPORTANT]
    > For m1 arch using fedora the following command

    ```bash
    firewall-cmd --add-port=11211/tcp --permanent
    firewall-cmd --reload
    sed -i 's/OPTIONS="-l 127.0.0.1"/OPTIONS=""/' /etc/sysconfig/memcached
    sudo systemctl restart memcached

    memcached -p 11211 -U 11111 -u memcached -d

    systemctl enable firewalld
    systemctl start firewalld
    systemctl status firewalld
    firewall-cmd --add-port=11211/tcp --permanent
    firewall-cmd --reload
    memcached -p 11211 -U 11111 -u memcache -d
    ```

5. Setup the RabbitMq server

    5.1 login into the RabbitMq vms

    ```bash
    vagrant ssh mc01
    ```

    5.2 switch to root user

    ```bash
    sudo -i
    ```

    5.3 update yum and accept using `-y` flag

    ```bash
    yum update -y
    ```

    5.4 install epel-release (this gives access to more packages)

    ```bash
    yum install epel-release -y
    ```

    > [!IMPORTANT]
    > For m1 arch use the following command:

    ```bash
    sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm -y

    sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

    setenforce 0
    ```

    5.5 Install RabbitMq dependencies

    ```bash
    curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | sudo bash
   
    yum clean all
   
    yum makecache
   
    yum install erlang -y
    ```

    5.6 Install RabbitMq server

    ```bash
    curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash

    yum install rabbitmq-server -y
    ```

    5.7 start, enable and check the status of the RabbitMq server

    ```bash
    systemctl start rabbitmq-server

    systemctl enable rabbitmq-server

    systemctl status rabbitmq-server
    systemctl is-enabled rabbitmq-server
    ```

    5.8 Setup user for RabbitMq

    ```bash
    sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'

    rabbitmqctl add_user test test

    rabbitmqctl set_user_tags test administrator
    ```

    - this setup a config file and then it update the file with the loopback_user info
  
    - created a test user
  
    - gave the test user permissions.

    > [!IMPORTANT]
    > For m1 arch using fedora the following command

    ```bash
    firewall-cmd --add-port=5671/tcp --permanent
    firewall-cmd --add-port=5672/tcp --permanent

    firewall-cmd --reload
    
    sudo systemctl restart rabbitmq-server
    ```

    exit and restart server

    ```bash
    sudo systemctl restart rabbitmq-server
    ```

    - Enabling the firewall and allowing port 25672 to access the rabbitmq permanently

    ```bash
    systemctl start firewalld
    
    systemctl enable firewalld
    
    firewall-cmd --get-active-zones
    
    firewall-cmd --zone=public --add-port=25672/tcp --permanent
    
    firewall-cmd --reload
    ```

6. Setup the Tomcat

    6.1 login into the database vms

    ```bash
    vagrant ssh mc01
    ```

    6.2 switch to root user

    ```bash
    sudo -i
    ```

    6.3 update yum and accept using `-y` flag

    ```bash
    yum update -y
    ```

    6.4 install epel-release (this gives access to more packages)

    ```bash
    yum install epel-release -y
    ```

    > [!IMPORTANT]
    > For m1 arch use the following command:

    ```bash
    sudo dnf install https://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm -y

    sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

    setenforce 0
    ```

    6.5 Install Tomcat dependencies

    ```bash
    yum install java-1.8.0-openjdk -y

    yum install git maven wget -y
    ```

    6.6 Download & extract Tomcat package

    ```bash
    cd /tmp/
    
    wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz

    tar xzvf apache-tomcat-8.5.37.tar.gz
    ```

    6.7 create a user for tomcat and give it necessary permissions

    ```bash
    useradd --home-dir /usr/local/tomcat8 --shell /sbin/nologin tomcat

    cp -r /tmp/apache-tomcat-8.5.37/* /usr/local/tomcat8/

    chown -R tomcat.tomcat /usr/local/tomcat8
    ```

    6.8 update the tomcat service and restart daemon

    6.9 start, enable and check the status of the tomcat service

    ```bash
    systemctl start tomcat

    systemctl enable tomcat

    systemctl status tomcat
    systemctl is-enabled tomcat
    ```

7. installing java project into Tomcat

    7.1 download from git

    ```bash
    git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git
    ```

    7.2 build code

    ```bash
    mvn install
    ```

    7.3 Deploy artifact

    ```bash
    systemctl stop tomcat

    rm -rf /usr/local/tomcat8/webapps/ROOT
   
    cp target/vprofile-v2.war /usr/local/tomcat8/webapps/ROOT.war
    
    systemctl start tomcat
    
    chown tomcat.tomcat usr/local/tomcat8/webapps -R
    
    systemctl restart tomcat
    ```

8. Setup the Nginx

    8.1 login into the database vms

    ```bash
    vagrant ssh web01
    ```

    8.2 switch to root user

    ```bash
    sudo -i
    ```

    8.3 update yum and accept using `-y` flag

    ```bash
    apt update

    apt upgrade
    ```

    8.4 install nginx

    ```bash
    apt install nginx -y
    ```

    8.5 Create Nginx conf file with below content

    `vi /etc/nginx/sites-available/vproapp`

    ```vim
    upstream vproapp {
        server app01:8080;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://vproapp;
        }
    }
    ```

    8.6 Remove default nginx conf and create a new link

    ```bash
    rm -rf /etc/nginx/sites-enabled/default
    ```

    Create link to activate website

    ```bash
    ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
    ```

    8.7 Restart Nginx

    ```bash
   systemctl restart nginx

   systemctl status nginx
   systemctl is-enabled nginx
   ```
