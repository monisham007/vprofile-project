# Vprofile
A multi-tier Java web application deployed locally using Vagrant-managed virtual machines. This project demonstrates a production-like stack with a load balancer, application server, caching layer, message broker, and relational database — all running on your local machine.

📐 Architecture Overview

1. The user navigates to the app via IP address in a browser.
2. NGINX acts as the load balancer and forwards the request to the Tomcat server.
3. Tomcat runs the Java web application and handles user authentication.
4. On login, the app queries MySQL for user credentials — but first checks Memcached.
5. Memcached acts as a database cache. On first login, the response is cached. Subsequent requests are served from cache, improving performance.
6. RabbitMQ is used as a message broker/queuing agent for decoupled service communication.
7. NFS provides shared external storage accessible across services.

<img width="809" height="508" alt="image" src="https://github.com/user-attachments/assets/41715a2e-86f6-441d-a20e-8bcbc66b32c9" />



🛠️ Tech Stack
| Service    | Role                          | VM Name |
|------------|-------------------------------|---------|
| NGINX      | Load Balancer / Reverse Proxy | `web01` |
| Tomcat     | Java Application Server       | `app01` |
| MySQL / MariaDB | Relational Database      | `db01`  |
| Memcache   | Database Caching              | `mc01`  |
| RabbitMQ   | Message Broker                | `rmq01` |
| NFS        | Shared Storage                | —       |


✅ Prerequisites
Before getting started, ensure you have the following installed:

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (or another Vagrant-compatible hypervisor)
- [Vagrant](https://developer.hashicorp.com/vagrant/downloads)
  Execute below command in your computer to install hostmanager plugin
  ```bash
  vagrant plugin install vagrant-hostmanager
  ```
- [VS Code](https://code.visualstudio.com/) with the **Vagrant** plugin installed
- Git

---


## 🚀 Getting Started

### 1. Clone the Repository

```bash
git clone <your-repo-url>
cd vprofile-project
```

Open the project in VS Code. Ensure the branch is set correctly:

```bash
git checkout main
# or switch from origin/local as needed
```

### 2. Start the Virtual Machines

Make sure **no other Vagrant VMs** are currently running to avoid conflicts.

```bash
cd  vagrant/Manual_provisioning
vagrant up
```

This will spin up all VMs defined in the `Vagrantfile`.

Setup should be done in below mentioned order:

MySQL (Database SVC) --> Memcache (DB Caching SVC) --> RabbitMQ (Broker/Queue SVC) --> Tomcat (Application SVC) --> Nginx (Web SVC)

### 3. Verify IP Mapping

Once the VMs are up, verify the IP mappings are correct by checking your hosts file or Vagrantfile. All services communicate via these internal IPs.

---

## ⚙️ Service Configuration

### 🔵 MySQL (MariaDB) — `db01`

```bash
vagrant ssh db01   # Login to the db vm
cat /etc/hosts     # Verify Hosts entry, if entries missing update the it with IP and hostnames
dnf update -y       #Update OS with latest patches
dnf install epel-release -y   #Set Repository
dnf install git mariadb-server -y      #Install Maria DB Package
systemctl start mariadb
systemctl enable mariadb        #Starting & enabling mariadb-server
```

Install and start MariaDB:

```bash
sudo systemctl start mariadb
sudo mysql_secure_installation  # Answer Yes to all prompts
```

Log in as root and run the setup SQL script to create the `accounts` database and populate it.

````bash
mysql -u root -padmin123  #Set DB name and users.
mysql> create database accounts;
mysql> grant all privileges on accounts.* TO 'admin'@'localhost' identified by
'admin123';
mysql> grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
mysql> FLUSH PRIVILEGES;
mysql> exit;
Download Source code & Initialize Database.
cd /tmp/
git clone -b local https://github.com/monisham007/vprofile-project.git
cd vprofile-project
mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
mysql -u root -padmin123 accounts
mysql> show tables;
mysql> exit;
systemctl restart mariadb   #Restart mariadb-server
````


### 🟡 Memcache — `mc01`

```bash
vagrant ssh mc01
cat /etc/hosts                       #Verify Hosts entry, if entries missing update the it with IP and hostnames
dnf update -y                        #Update OS with latest patches
sudo dnf install epel-release -y     #Install, start & enable memcache on port 11211
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached

```

Install Memcache and update the configuration to listen on the correct port/interface. Note the difference in the config file **before and after** running the configuration command — Memcache should be bound to `0.0.0.0` to allow remote connections.


### 🟠 RabbitMQ — `rmq01`

```bash
vagrant ssh rmq01  #Login to the RabbitMQ vm
cat /etc/hosts    #Verify Hosts entry, if entries missing update the it with IP and hostnames
dnf update -y  #Update OS with latest patches
dnf install epel-release -y #Set EPEL Repository
#Install Dependencies
sudo dnf install wget -y
dnf -y install centos-release-rabbitmq-38
dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
systemctl enable --now rabbitmq-server
#Setup access to user test and make it admin
sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
sudo systemctl restart rabbitmq-server
```


### 🔴 Tomcat — `app01`
First setup systemctl for tomcat 
```bash
#Download tomcat
wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.28/bin/apache-tomcat-10.1.28.tar.gz

#Extract tomcat
tar xzvf apache-tomcat-10.1.28.tar.gz

#Create tomcat user
useradd --home-dir /opt/tomcat --shell /sbin/nologin tomcat

#Copy files to tomcat home directory
cp -r apache-tomcat-10.1.28/* /opt/tomcat/

#Give tomcat user ownership to tomcat directory
chown -R tomcat.tomcat /opt/tomcat/


#Create system file for tomcat service
vim /etc/systemd/system/tomcat.service

#Paste below content

[Unit]
Description=Tomcat
After=network.target


[Service]
Type=forking

User=tomcat
Group=tomcat

WorkingDirectory=/opt/tomcat

Environment=JAVA_HOME=/usr/lib/jvm/jre

Environment=CATALINA_HOME=/opt/tomcat
Environment=CATALINE_BASE=/opt/tomcat

ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target


# Reload system config changes
systemctl daemon-reload

#Start & Enable tomcat service
systemctl start tomcat
systemctl status tomcat
systemctl enable tomcat
```
To set up app01

````
#Login to the tomcat vm
vagrant ssh app01

#Verify Hosts entry, if entries missing update the it with IP and hostnames
cat /etc/hosts

#Update OS with latest patches
dnf update -y

#Set Repository
dnf install epel-release -y

#Install Dependencies
dnf -y install java-17-openjdk java-17-openjdk-devel
dnf install git wget -y

#Change dir to /tmp
cd /tmp/

#Download & Tomcat Package
wget
https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10
.1.26.tar.gz
tar xzvf apache-tomcat-10.1.26.tar.gz

#Add tomcat user
useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat

#Copy data to tomcat home dir
cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/

#Make tomcat user owner of tomcat home dir
chown -R tomcat.tomcat /usr/local/tomcat

#Setup systemctl command for tomcat as shown above

#Create tomcat service file
vi /etc/systemd/system/tomcat.service

#Update the file with below content
[Unit]
Description=Tomcat
After=network.target
[Service]
User=tomcat
Group=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINE_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
RestartSec=10
Restart=always
[Install]
WantedBy=multi-user.target

#Reload systemd files
systemctl daemon-reload
#Start & Enable service
systemctl start tomcat
systemctl enable tomcat

````
The application config file is located in the Tomcat configuration directory. Key notes:

- Tomcat deploys the application from **`ROOT.war`** — a standard Java WAR archive.
- On first deploy, `ROOT.war` is automatically extracted to a `ROOT/` directory.
- The `ROOT/` folder contains the expanded application files.

```
/usr/local/tomcat/webapps/
├── ROOT.war        ← original archive
└── ROOT/           ← extracted application (auto-generated)
```

Update the `application.properties` inside the config directory with correct service hostnames/IPs before deploying.
At this stage, the source code has been successfully downloaded and is available in the local repository. However, the application cannot be run directly on a Tomcat server in its current form. The source code must first be compiled and packaged into a deployable artifact that Tomcat can recognize and execute.

To perform the compilation and packaging process, a build automation tool called Maven is required. Maven manages project dependencies, compiles the source code, and generates the deployable package (typically a WAR file). The immediate objective is simply to install Maven, download the source code, and build the application.

Once the build process is completed using Maven commands, the resulting artifact can be deployed to the Tomcat server. The deployment process involves removing the default Tomcat application, copying the newly generated artifact into the Tomcat deployment directory, and then starting or restarting the Tomcat service. After these steps are completed, Tomcat will deploy and host the application.

MAVEN setup
````
cd /tmp/
wget
https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3
.9.9-bin.zip
unzip apache-maven-3.9.9-bin.zip
cp -r apache-maven-3.9.9 /usr/local/maven3.9
export MAVEN_OPTS="-Xmx512m"

#Download Source code
git clone -b local https://github.com/hkhcoder/vprofile-project.git

#Update configuration
cd vprofile-project
vim src/main/resources/application.properties
# Update file with backend server details
````
Run below command inside the repository (vprofile-project)
````
/usr/local/maven3.9/bin/mvn install
#Deploy artifact
systemctl stop tomcat
rm -rf /usr/local/tomcat/webapps/ROOT*
cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
systemctl start tomcat
chown tomcat.tomcat /usr/local/tomcat/webapps -R
systemctl restart tomcat
````

### 🟢 NGINX — `web01`

```bash
#Login to the Nginx vm
vagrant ssh web01
sudo -i

#Verify Hosts entry, if entries missing update the it with IP and hostnames
cat /etc/hosts

#Update OS with latest patches
apt update
apt upgrade

#Install nginx
apt install nginx -y

#Create Nginx conf file
vi /etc/nginx/sites-available/vproapp

#Update with below content
upstream vproapp {
server app01:8080;
}
server {
listen 80;
location / {
proxy_pass http://vproapp;
}
}

#Remove default nginx conf
rm -rf /etc/nginx/sites-enabled/default

#Create link to activate website
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp

#Restart Nginx
systemctl restart nginx
```

---

## 🌐 Accessing the Application

Once all services are running, open a browser and navigate to:

```
http://<web01-ip>/
```

You should see the Vprofile login page. Use the credentials seeded into the MySQL database during setup.

<img width="808" height="410" alt="image" src="https://github.com/user-attachments/assets/734e41b3-3095-423d-9646-f5d3b379be46" />
<img width="809" height="407" alt="Screenshot 2026-06-09 180511" src="https://github.com/user-attachments/assets/a1327772-e3bf-4a1e-bf0e-b93eebbb88d1" />

All Users

<img width="810" height="407" alt="image" src="https://github.com/user-attachments/assets/9658dbe8-4d53-470d-8bd8-389b840dcdd7" />

When user logs in for the first time then data comes from DB then stored in cache.

<img width="806" height="404" alt="Screenshot 2026-06-09 181659" src="https://github.com/user-attachments/assets/e6d9177c-5d68-450a-a9bb-f9b7be413ff4" />

When same user is clicked again, the data is retrieved from memcache.

<img width="809" height="395" alt="Screenshot 2026-06-09 182213" src="https://github.com/user-attachments/assets/f42f85cd-4d4f-4289-83e6-39cac2e6bb25" />


---

## 📁 Project Structure

```
vprofile-project/
├── vagrant/
│   └── Vagrantfile          # VM definitions and network config
├── src/
│   └── main/
│       ├── java/            # Java application source
│       └── resources/
│           └── application.properties  # DB, cache, MQ config
├── target/
│   └── vprofile-v2.war      # Built WAR artifact
└── README.md
```

---

## 🔍 How Memcache Works in This Project

On first login:
1. App queries Memcache → **cache miss**
2. App queries MySQL → fetches user data
3. Data is returned to Tomcat **and** stored in Memcache

On subsequent logins (same user):
1. App queries Memcache → **cache hit**
2. Data returned directly from cache — **no DB query needed**

This mimics the behaviour of browser or CDN caching, but at the database layer.

---

## 🐛 Troubleshooting

| Issue | Resolution |
|---|---|
| VMs fail to start | Ensure no other VMs are running; check VirtualBox for conflicts |
| Cannot connect to DB | Verify MariaDB is running: `systemctl status mariadb` |
| App not loading | Check Tomcat logs: `catalina.out` in Tomcat's `logs/` directory |
| Memcache not caching | Confirm it's bound to `0.0.0.0`, not `127.0.0.1` |
| NGINX 502 Bad Gateway | Ensure Tomcat is running on port 8080 and the proxy config is correct |

---

## 📚 Key Concepts

- **WAR file**: Web Application Archive — a packaged Java web app deployed to Tomcat.
- **Memcache**: An in-memory key-value store used to cache database query results.
- **RabbitMQ**: A message broker that enables async communication between services.
- **NFS**: Network File System — allows VMs to share a common storage volume.
- **Reverse Proxy**: NGINX forwards client requests to the backend Tomcat server, hiding internal topology.

---

