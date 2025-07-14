# *How to setup SonarQube server and Intergrate with jenkins job*
-  SonarQube can be integrated into CI/CD pipelines to ensure code quality is mantained throughout the development lifecycle.
-  It provides detailed metrics and reports on various aspects of code quality, such as code complexity, duplication, test coverage, and documentation.
#### To install and configure sonarqube
-  sonarqube requires java -> to run
-  sonarqube supports various databases like PostreSQL, MySQL, Oracle and Microsoft SQL Server etc.
-  PostreSQL is a common choice.
#### SonarQube required minimum of:
-  CPU - 2+ Cores
-  RAM 2+ GB
-  Disk space: 1+ GB free space
## How to set-up the SonarQube server?
let me drive you in detail:
-  Go to sonarsource.com website (sonarqube is a product of sonar source)
-  Navigate to docs -> find 'server installation and setup' -> sonarqube server host -> see the requirements
-  Launch ec2 instance with required configurations.
### Login to server & Start the installations :
1.  **Install Java**
```bash
apt update 
apt install openjdk-17-jdk -y
java -version
```
2.  **To store & organize all the data & reports it uses any database**
```bash
apt update
apt install postgresql postgresql-contrib -y 
systemctl start postgresql 
systemctl enable postgresql
```
3.  **Login as (by default a user is created) postgres user & do necessary setups**
```bash
su - postgres
```
**Run the below command to create a new user called "sonar"**
```bash
createuser sonar
```
**Switch to sql shell**
```bash
psql
```
**Set the password here**
```bash
ALTER USER sonar WITH ENCRYPTED password 'password';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
```
-  To exit from postgres SQL use: `\q`
-  Type `exit` to logout of postgres user

## Lets dowlaod the sonarqube binary file:
```bash
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.8.100196.zip
```
**Unzip the file:**
```bash
#install unzip tool `apt install unzip`
unzip sonarqube-9.9.8.100196.zip -d /opt/
```
**Rename the unzipped directiory:**
```bash
mv sonarqube-9.9.8.100196/ sonarqube -v
```
**Creat a group :**
```bash
groupadd sonarGroup
```
**Create a user sonar with /opt/sonarqube as home directory and primary group as sonarGroup:**
```bash
useradd -c "user to run sonarqube" -d /opt/sonarqube/ -g sonarGroup sonar
```
**Change ownership on directory:**
```bash
chown -R sonar:sonarGroup /opt/sonarqube
```
### Need to edit few file to start the sonarqube service:
```bash
cd /opt/sonarqube/conf
nano sonar.properties
```
**Edit the below line from the `sonar.properties` file:**
```bash
sonar.jdbc.username=sonar
sonar.jdbc.password=password
```
**Postgres sql server will communicate on this url :** (copy paste existing line and edit as mentioned)
```bash
#sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube?currentSchema=my_schema
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
```
**By default sonar web will listen on port 9000, if you want to customize it give desired port number**
```bash
#sonar.web.port=9000
```
### Running SonarQube Server as a service on Linux with systemd :
**Create a service file to achieve the goal**
```bash
vim /etc/systemd/system/sonarqube.service
```
**Add the below lines:**
```text
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
User=sonar
Group=sonarGroup
PermissionsStartOnly=true
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
StandardOutput=journal
LimitNOFILE=131072
LimitNPROC=8192
Restart=always

[Install]
WantedBy=multi-user.target
```
### Specify those limits inside your unit file in the section [to set service for systemd user] : 
```bash
vim sonarqube/bin/linux-x86-64/sonar.sh
```
**Add this line line**
```bash
RUN_AS_USER=sonar
```
**Set limits** (at the end -add the below lines)
```bash
vim /etc/sysctl.conf
```
```bash
vm.max_map_count=262144
fs.file-max=65536
```
Edit :
```bash
vim /etc/security/limits.conf
```
At the end- add the below:
```bash
sonar - nofile 65536
sonar - nproc 4096
```
**To see the applied changes, if applied properly:**
```bash
sysctl -p
```
It should show:
```bash
vm.max_map_count=262144
fs.file-max=65536
```
### Reload & start the SonarQube service
```bash
systemctl daemon-reload
systemctl enable sonarqube.service
systemctl start sonarqube.service
systemctl status sonarqube.service
```
**Check for the sonar.logs**
```bash
tail -f /opt/sonarqube/logs/sonar.log
```
**you should be seeing this message as sucess note of sonarqube**
```bash
Process[ce] is up
SonarQube is operational
```
### Now connect to sonarqube web UI : 
-  <Server-ip>:9000
-  default login credentails :
```bash
username : admin
password: admin
```
**-> it will prompt to set new password -> set a secure password & your SonarQube Web UI is all set to use**
