# Plan: Jenkins & Tomcat installation and intergration

`We'll install everything from scratch and set up remote deployment from Jenkins to Tomcat using the Tomcat Manager API.`
-  **Server A → Jenkins Server**
-  **Server B → Tomcat Server**

## Prerequisites on Both Servers

-  Ubuntu/Debian (or similar)
-  Root or sudo access
-  OpenJDK installed
-  Internet access

## Part 1: Jenkins Setup (on Server A)

1.1  Install Java
```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

1.2  Install Jenkins
```bash
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```
1.3  Unlock Jenkins & Install Plugins

-  Access Jenkins from browser:
`http:<server1-ipv4>:8080`

-  Get initialAdminPassword:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then:
-  Complete setup
-  Install suggested plugins
-  Create admin user or create a user and its totally optional as you have an option to create user later.

1.4  Install “Deploy to container” Plugin

-  Go to: Manage Jenkins → Manage Plugins
-  Under Available tab:
-  `Search and install: Deploy to container Plugin, Publish Over SSH`
-  Restart Jenkins

## Where Should Maven Be Installed?
### Maven should be installed on the Jenkins server, not on the Tomcat server.
**Why?**
-  Jenkins is responsible for building the application from source code (e.g., compiling, testing, packaging)
-  Maven is a build tool, not a deployment tool.
-  Once Jenkins uses Maven to produce a .war file, it then deploys that .war to the remote Tomcat server.

### Jenkins Server (Build + CI/CD)
Install Maven here:
```bash
sudo apt update
sudo apt install maven -y
mvn -version
```
Jenkins uses Maven to run commands like:
```bash
mvn clean package
```
Which generates:
```bash
target/yourapp.war
```
This we will define in detail while setting a job or event build steps.

#### Tomcat Server (Deployment Only)
-  Tomcat only needs Java (to run)
-  It receives the .war file from Jenkins over HTTP via the Tomcat Manager API
-  No Maven required here

## Part 2: Tomcat Setup (on Server B)

2.1  Install Java

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```
Note: It is highly recommanded to match the java version on both Jenkins & tomcat server.

2.2  Download & Configure Tomcat
```bash
cd /opt
sudo wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.107/bin/apache-tomcat-9.0.107.tar.gz
sudo tar -xvzf apache-tomcat-9.0.107.tar.gz
sudo mv apache-tomcat-9.0.107 tomcat9
```
Create a user account:
```bash
useradd -M -d /opt/tomcat9 tomcat
```
Check the ownership over the directory /opt/tomcat9
```bash
ls -ld /opt/tomcat9/
```
It will be showing like this:
```bash
drwxr-xr-x 9 root root 4096 Jul 12 16:10 /opt/tomcat9/
```
You need to change the ownership, modify the permissions and check once again :
```bash
chown -R tomcat:tomcat /opt/tomcat9/
sudo chmod +x /opt/tomcat9/bin/*.sh
ls -ld /opt/tomcat9
```
Verify user account created with /opt/tomcat9 directory as a home directory
```bash
grep tomcat /etc/passwd
```
It should show like this:
```bash
tomcat:x:1001:1001::/opt/tomcat9:/bin/sh
```
2.3  Create a service file in:
```bash
cd /usr/lib/systemd/system
vim tomcat.service
```
You need to add the below lines:
```bash
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
Type=oneshot
ExecStart=/opt/tomcat9/bin/startup.sh
ExecStop=/opt/tomcat9/bin/shutdown.sh
RemainAfterExit=yes
User=tomcat
Group=tomcat

[Install]
WantedBy=multi-user.target
```

2.4  Configure Tomcat User for Remote Deployment
```bash
sudo nano /opt/tomcat9/conf/tomcat-users.xml
```
Add inside <tomcat-users>:

```xml
<role rolename="manager-gui"/>
<role rolename="admin-gui"/>
<user username="manager" password="managerpass123" roles="manager-gui"/>
<user username="admin" password="adminpass123" roles="admin-gui"/>
<user username="deployer" password="deploypass123" roles="manager-script"/>
```
2.5  Allow Remote Deployment
-  Edit conf/web.xml or use context files in webapps/manager/META-INF/context.xml:
```bash
sudo nano /opt/tomcat9/webapps/manager/META-INF/context.xml
```
Comment or remove the IP access restriction:

```xml<!--
<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.\d+\.\d+\.\d+|::1" />
-->
```
Restart tomcat:
```bash
systemctl daemon-reload
systemctl restart tomcat.service
systemctl status tomcat.service
```

2.6  Start Tomcat
```bash
/opt/tomcat/bin/startup.sh
```
Verify:
http://tomcatserver-ip:8080
> Login with: manager / managerpass123

#### Summary
| Component | Jenkins Server | Tomcat Server |
| --------- | -------------- | ------------- |
| Java      | Required       | Required    |
| Maven     |  Required     |  Not Required   |
| Tomcat    |  Not Required   |  Required    |
| Jenkins   |  Required     |  Not Required  |


## Part 3: Integration (Deploy WAR from Jenkins to Tomcat)

3.1.  Create Jenkins Job
-  Before creating the job -> install required plugins: i.Deploy to containers ii.Publish Over SSH
-  Go to Manage Jenkins -> Systems -> Search for Publish Over SSH & fill the following:
```bash
Key : tomact server pvt key text
-> Add server
Name : TomcatOverSSH (your choice)
Hostname: tomcat server ipv4
Username: ubuntu
Remote Directory : /home/ubuntu
-> test configuration -> sucess (expected)
```
-  Jenkins Dashboard -> New Item -> Freestyle Project
-  Add Git repo (if using)
-  In Build, Invoke top-level Maven targets :
```bash
clean install
```
2.  Add Post-build Action -> Publish Over SSH
```bash
Transfers : source files - **/*.war
remote directory : tomcat
```
exec command: 
```bash
sudo cp /home/ubuntu/tomcat/*.war /opt/tomcat9/webapps
```
4.  Fill:
-  WAR/EAR files: target/yourapp.war
-  Container: Tomcat 10.x
-  Manager URL: http://Server-B-ip:8080/manager/text
4.  Credentials:
-  Click Add → Jenkins
-  Username: deployer
-  Password: deploypass123

## Part 4: Test the Pipeline

1.  Click Build Now
2.  Observe the console output & track the build status.\
3.  Build sucess -> go to tomcat & check:
-  Go to: http://192.168.1.20:8080/yourapp
-  Jenkins pushes .war to Tomcat via remote API
