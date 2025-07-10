# Plan: Jenkins & Tomcat installation and intergration

-  We'll install everything from scratch and set up remote deployment from Jenkins to Tomcat using the Tomcat Manager API.

> Server A → Jenkins Server
> Server B → Tomcat Server

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
-  `Search and install: Deploy to container Plugin`
-  Restart Jenkins

## Part 2: Tomcat Setup (on Server B)

2.1  Install Java

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```
Note: It is highly recommanded to match the java version on both Jenkis & tomcat server.

2.2  Download & Configure Tomcat
```bash
cd /opt
sudo wget https://downloads.apache.org/tomcat/tomcat-10/v10.1.16/bin/apache-tomcat-10.1.16.tar.gz
sudo tar -xvzf apache-tomcat-10.1.16.tar.gz
sudo mv apache-tomcat-10.1.16 tomcat
sudo chmod +x /opt/tomcat/bin/*.sh
```

2.3  Configure Tomcat User for Remote Deployment
```bash
sudo nano /opt/tomcat/conf/tomcat-users.xml
```
Add inside <tomcat-users>:

```xml
<role rolename="manager-script"/>
<user username="deployer" password="deploypass123" roles="manager-script"/>
```
2.4  Allow Remote Deployment
Edit conf/web.xml or use context files in webapps/manager/META-INF/context.xml:

```bash
sudo nano /opt/tomcat/webapps/manager/META-INF/context.xml
```
Comment or remove the IP access restriction:

```xml<!--
<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.\d+\.\d+\.\d+|::1" />
-->
```

2.5  Start Tomcat
```bash
/opt/tomcat/bin/startup.sh
```
Verify:
http://192.168.1.20:8080/manager/html
> Login with: deployer / deploypass123

## Part 3: Integration (Deploy WAR from Jenkins to Tomcat)

3.1  Create Jenkins Job

-  Jenkins Dashboard -> New Item -> Freestyle Project
-  Add Git repo (if using)
-  In Build, use Maven to generate .war:
```bash
mvn clean package
```
1.  Add Post-build Action -> Deploy WAR/EAR to a container
2.  Fill:
   2.1  WAR/EAR files: target/yourapp.war
-    Container: Tomcat 10.x
-    Manager URL: http://192.168.1.20:8080/manager/text

-  Credentials:
-    Click Add → Jenkins
-    Username: deployer
-    Password: deploypass123

## Part 4: Test the Pipeline

-  Click Build Now
-  Observe the console output & track the build status.\
-  Build sucess -> go to tomcat & check:
-    Go to: http://192.168.1.20:8080/yourapp
-    Jenkins pushes .war to Tomcat via remote API
