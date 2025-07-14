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
3.  **Login as (by default a user is created) postgres user:**
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
**set the password here**
```bash
ALTER USER sonar WITH ENCRYPTED password 'password';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
```
-  To exit from postgres SQL use: `\q`
-  Type `exit` to logout of postgres user
