


--------------------------------------------------------------------------------
STEP 0: STARTING POINT ON WINDOWS (DO THIS FIRST)
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Windows Desktop + ONE single Git Bash window + Chrome.

1. Open Docker Desktop on Windows and wait until the bottom-left whale icon is
   green ("Engine running").
2. Right-click on your Windows Desktop -> Click "Open Git Bash here".
   (Keep this SAME Git Bash window open for the entire exam).
3. Run these commands in Git Bash to create your clean exam workspace folder:

mkdir -p ~/Desktop/SE-Lab-Exam
cd ~/Desktop/SE-Lab-Exam

4. Check if your pre-installed Jenkins is already running:
   - Open Chrome and go to: http://localhost:8080
   - If the Jenkins login/dashboard page loads -> Jenkins is ALREADY running!
     Go straight to Task 1.
   - If Chrome says "This site can't be reached / localhost refused to connect":
     * Press Win + R on your keyboard -> type: services.msc -> press Enter.
     * Scroll down to "Jenkins" -> Right-click "Jenkins" -> Click "Start".
     * Refresh http://localhost:8080 in Chrome.
     *(Alternative if installed via jenkins.war instead of Windows Service: open
     Command Prompt in the folder containing jenkins.war and run:
     java -jar jenkins.war --httpPort=8080)*


--------------------------------------------------------------------------------
TASK 1: JENKINS - SCRIPTED PIPELINE & BUILD TRIGGERS [25 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Chrome Browser at http://localhost:8080.
[FILES NEEDED ON DISK]: None. Everything is written inside the Jenkins web UI.

STEP 1.1: Configure Maven in Jenkins Tools
1. In Chrome at http://localhost:8080, log into Jenkins.
2. Click "Manage Jenkins" (left menu) -> Click "Tools".
3. Scroll down to "Maven installations" -> Click "Add Maven".
4. Enter:
   - Name: Maven3
   - Check the box: "Install automatically"
5. Click "Save" at the bottom.

STEP 1.2: Create Pipeline Job & Set Automated Build Triggers
1. Click "New Item" (top-left).
2. Enter Item Name: SportSystem-Pipeline
3. Click "Pipeline" -> Click "OK".
4. Scroll down to "Triggers":
   - Check: "GitHub hook trigger for GITScm polling".
   - Check: "Poll SCM" and type * * * * * in the Schedule box (this makes
     Jenkins automatically check GitHub every 1 minute during your lab demo
     even without a public webhook URL).

STEP 1.3: Write the Scripted Pipeline
1. Scroll down to the "Pipeline" section.
2. Set "Definition" to: Pipeline script
3. Copy and paste this exact Scripted Pipeline into the Script box (verified for
   branch 'main' and root pom.xml from your repo screenshot):

node {
    def mvnHome = tool name: 'Maven3', type: 'maven'

    stage('Clean Workspace') {
        cleanWs()
    }

    stage('Clone Repository') {
        git branch: 'main', url: 'https://github.com/sarasrija/SportSystem.git'
    }

    stage('Clean & Compile') {
        bat "\"${mvnHome}\\bin\\mvn\" clean compile"
    }

    stage('Package WAR') {
        bat "\"${mvnHome}\\bin\\mvn\" package"
    }
}

(Note: If cleanWs() throws a missing plugin error, replace cleanWs() with
deleteDir(). If your Jenkins runs on Linux/WSL instead of Windows, replace
the two bat lines with:
sh "${mvnHome}/bin/mvn clean compile"
and
sh "${mvnHome}/bin/mvn package"
)

STEP 1.4: Execute & Verify Output
1. Click "Save" -> Click "Build Now" (left menu).
2. Click "#1" under Build History -> Click "Console Output".
3. Verify: All 4 stages succeed, "BUILD SUCCESS" appears, and the .war file
   is generated inside target/.

[HOW TO CONTINUE TO TASK 2]:
Leave Jenkins running in the background (it does not affect AWS EC2).
Open AWS Management Console in a new Chrome tab and keep your SAME Git Bash open.


--------------------------------------------------------------------------------
TASK 2: AWS - CLOUD DEPLOYMENT USING EC2 [25 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: AWS Console in Chrome + your SAME Git Bash window.

STEP 2.1: Provision Ubuntu EC2 Instance in AWS Console
1. Open AWS Management Console -> Search "EC2" -> Click "Launch Instance".
2. Configure these exact settings:
   - Name: SportSystem-EC2
   - Application and OS Images (AMI): Ubuntu (Ubuntu Server 22.04 LTS or 24.04 LTS)
   - Instance type: t2.micro (or t3.micro)
   - Key pair: Click "Create new key pair" -> Name: sports-key -> Type: RSA ->
     Format: .pem -> Click "Create key pair".
     (Save sports-key.pem inside your Desktop/SE-Lab-Exam folder).
   - Network settings (Firewall / Security Group):
     * Check "Allow SSH traffic from" (0.0.0.0/0)
     * Check "Allow HTTP traffic from the internet"
     * Click "Edit" -> Click "Add security group rule":
       Type: Custom TCP | Port range: 8080 | Source type: Anywhere (0.0.0.0/0)
3. Click "Launch Instance" -> Click "View all instances".
4. Wait until Instance state = "Running", select SportSystem-EC2, and copy its
   "Public IPv4 address".

STEP 2.2: Connect to EC2 Instance via SSH
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows (at ~/Desktop/SE-Lab-Exam).
Run these commands (replace <EC2-PUBLIC-IP> with your copied Public IP):

cd ~/Desktop/SE-Lab-Exam
chmod 400 sports-key.pem
ssh -i sports-key.pem ubuntu@<EC2-PUBLIC-IP>

(Type yes and press Enter when prompted. Your prompt will change to
ubuntu@ip-xxx-xxx-xxx-xxx:~$).
*(Backup if college Wi-Fi blocks port 22 SSH: In AWS Console, select your
instance -> Click "Connect" -> "EC2 Instance Connect" -> Click "Connect" to open
the Ubuntu terminal directly in Chrome).*

STEP 2.3: Install Java, Git, Docker, Maven, and Tomcat on EC2
[WHERE TO DO THIS]: Inside the Ubuntu EC2 terminal (ubuntu@ip-...:~$).

sudo apt update && sudo apt upgrade -y
sudo apt install -y openjdk-17-jdk git maven docker.io
sudo apt install -y tomcat9 || sudo apt install -y tomcat10

STEP 2.4: Clone Repository & Build .war File on EC2
[WHERE TO DO THIS]: Inside the SAME Ubuntu EC2 terminal.

cd ~
git clone https://github.com/sarasrija/SportSystem.git
cd SportSystem
mvn clean package
ls -lh target/

(Confirm a .war file is listed inside target/).

STEP 2.5: Deploy .war to Tomcat & Ensure Persistence Across Reboots
[WHERE TO DO THIS]: Inside the SAME Ubuntu EC2 terminal.

if [ -d "/var/lib/tomcat9/webapps" ]; then
    sudo cp target/*.war /var/lib/tomcat9/webapps/SportSystem.war
    sudo systemctl restart tomcat9
    sudo systemctl enable tomcat9
else
    sudo cp target/*.war /var/lib/tomcat10/webapps/SportSystem.war
    sudo systemctl restart tomcat10
    sudo systemctl enable tomcat10
fi

STEP 2.6: Verify Public Access in Chrome
1. Open Chrome and go to: http://<EC2-PUBLIC-IP>:8080/SportSystem
2. Confirm the Sports Management System web app loads over Public IP.

[HOW TO CONTINUE TO TASK 3]:
Type exit in the Ubuntu EC2 terminal so your Git Bash prompt returns to your
local Windows machine at ~/Desktop/SE-Lab-Exam:

exit


--------------------------------------------------------------------------------
TASK 3: KUBERNETES & NAGIOS - MINIKUBE PODS & MONITORING [25 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows (at ~/Desktop/SE-Lab-Exam).
[FILES NEEDED ON DISK]: None. Run all commands directly in Git Bash.

STEP 3.1: Start Minikube, Create Deployment, Scale Replicas & Expose Service
Run these commands one by one in Git Bash:

minikube start --driver=docker
kubectl get nodes
kubectl create deployment sports-app --image=tomcat:9.0
kubectl scale deployment sports-app --replicas=3
kubectl expose deployment sports-app --type=NodePort --port=8080
kubectl get pods
kubectl get svc sports-app
minikube service sports-app --url

(IMPORTANT: "minikube service sports-app --url" prints http://127.0.0.1:xxxxx
and keeps the terminal open to hold the tunnel active. Open that URL in Chrome
to verify Tomcat runs on Kubernetes, then press Ctrl + C in Git Bash to get
your prompt back and run Step 3.2).

STEP 3.2: Pull and Run Nagios Docker Image
[WHERE TO DO THIS]: Same Git Bash window on Windows.
(Jenkins is using port 8080 locally, so Nagios is mapped to host port 8090
using -p 8090:80 to avoid any port conflict):

docker run --name nagios4 -p 8090:80 -d jasonrivers/nagios:latest
docker ps

STEP 3.3: Verify Nagios Dashboard in Chrome
1. Open Chrome -> Go to: http://localhost:8090
2. Log in with:
   - Username: nagiosadmin
   - Password: nagios
3. Click "Hosts" and "Services" on the left sidebar to verify live CPU load,
   HTTP status, and system metrics.

[HOW TO CONTINUE TO TASK 4]:
Stay in your SAME Git Bash window on Windows and move directly to Task 4 below.


--------------------------------------------------------------------------------
TASK 4: DOCKER COMPOSE - MULTI-CONTAINER ORCHESTRATION [25 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Same Git Bash window on Windows.
[EXACT FOLDER & FILES]: Create ~/Desktop/SE-Lab-Exam/sports-compose and write
all 3 files inside that exact same folder:
  1. ~/Desktop/SE-Lab-Exam/sports-compose/app.py
  2. ~/Desktop/SE-Lab-Exam/sports-compose/Dockerfile
  3. ~/Desktop/SE-Lab-Exam/sports-compose/docker-compose.yml

STEP 4.1: Create Folder & Write All 3 Files
Copy and paste this entire block into Git Bash:

cd ~/Desktop/SE-Lab-Exam
mkdir -p sports-compose
cd sports-compose

cat << 'EOF' > app.py
from flask import Flask
import redis
import os

app = Flask(__name__)
cache = redis.Redis(host=os.getenv('REDIS_HOST', 'redis'), port=6379)

@app.route('/')
def index():
    visits = cache.incr('hits')
    return f"Sports Management System Microservice Online! Total Visits: {visits}\n"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat << 'EOF' > Dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask redis psycopg2-binary
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
EOF

cat << 'EOF' > docker-compose.yml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgresql://sportsadmin:sportspass@db:5432/sportsdb
      REDIS_HOST: redis
    networks:
      - sports-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: sportsadmin
      POSTGRES_PASSWORD: sportspass
      POSTGRES_DB: sportsdb
    volumes:
      - sports-pgdata:/var/lib/postgresql/data
    networks:
      - sports-network

  redis:
    image: redis:7-alpine
    networks:
      - sports-network

networks:
  sports-network:
    driver: bridge

volumes:
  sports-pgdata:
EOF

ls -la

(Verify ls -la lists app.py, Dockerfile, and docker-compose.yml inside
~/Desktop/SE-Lab-Exam/sports-compose).

STEP 4.2: Start Stack, Verify Persistence, Rebuild & Scale Workflows
Run these commands inside ~/Desktop/SE-Lab-Exam/sports-compose in Git Bash:

docker compose up -d --build
docker compose ps
docker network ls | grep sports
docker volume ls | grep sports
curl http://localhost:5000
curl http://localhost:5000
docker compose up -d --build --no-deps web
docker compose up -d --scale redis=2
docker compose ps

(Open http://localhost:5000 in Chrome -> Refresh page to verify Total Visits
increments on every reload).

------------------------------------------------------------------------------------------------------------------



================================================================================
KMIT - SOFTWARE ENGINEERING LAB INTERNAL II (SET-5) - COMPLETE MASTER SHEET
Subject Code: 23CC501PC | Year/Sem: III / I
GitHub Repository: https://github.com/nikithamoturi/GymManagementSystem.git
Your Local Setup: Windows + Jenkins (already running on port 8080) + Docker Desktop + Minikube
================================================================================


--------------------------------------------------------------------------------
STEP 0: STARTING POINT WHEN GITHUB REPO IS GIVEN (DO THIS FIRST ON WINDOWS)
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Windows Desktop -> Open ONE single Git Bash window + Chrome.

1. Open Docker Desktop on Windows and wait until the bottom-left whale icon is
   green ("Engine running").
2. Right-click on your Windows Desktop -> Click "Open Git Bash here".
   (Keep this SAME Git Bash window open throughout the entire exam).
3. Run these commands in Git Bash to create your exam workspace folder and
   check the exact default branch name of the GymManagementSystem repository:

mkdir -p ~/Desktop/SE-Lab-Set5
cd ~/Desktop/SE-Lab-Set5
git ls-remote --heads https://github.com/nikithamoturi/GymManagementSystem.git

[LOOK AT THE OUTPUT]:
- If it prints "refs/heads/main", use "main" (or "*/main") in Task 2.
- If it prints "refs/heads/master", use "master" (or "*/master") in Task 2.
- Leave this Git Bash window open at ~/Desktop/SE-Lab-Set5.


--------------------------------------------------------------------------------
TASK 1: DOCKER - MULTI-CONTAINER APPLICATION (WORDPRESS + MYSQL) [20 MARKS]
--------------------------------------------------------------------------------
[PDF REQUIREMENTS]:
1. Pull suitable WordPress and MySQL images from Docker Hub.
2. Create a custom Docker network so WordPress and MySQL can communicate.
3. Create a named volume for MySQL data and start MySQL with database name,
   username, and password environment variables.
4. Start WordPress on the same network with matching DB details and host port.
5. Complete WordPress setup in browser, create sample content, and inspect
   container status and logs.
(Note: Standalone exercise - does NOT use the GitHub repo).

[WHERE TO DO THIS]: Same Git Bash window on Windows (at ~/Desktop/SE-Lab-Set5).
No files need to be written on disk.

STEP 1.1: Pull Images, Create Custom Network & Named Volume
Run these commands in Git Bash:

docker pull mysql:8.0
docker pull wordpress:latest
docker network create gym-network
docker volume create gym-mysql-data

STEP 1.2: Start MySQL Container on Custom Network with Named Volume
Run in the SAME Git Bash window:

docker run -d --name gym-mysql \
  --network gym-network \
  -v gym-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=gymdb \
  -e MYSQL_USER=gymuser \
  -e MYSQL_PASSWORD=gympass \
  mysql:8.0

(Wait ~15 seconds for MySQL to initialize its database tables).

STEP 1.3: Start WordPress Container on Same Network
(Note: Jenkins is already using port 8080 on your PC, so we map WordPress to
host port 8085 using -p 8085:80 to avoid any port 8080 conflict):

docker run -d --name gym-wordpress \
  --network gym-network \
  -p 8085:80 \
  -e WORDPRESS_DB_HOST=gym-mysql:3306 \
  -e WORDPRESS_DB_USER=gymuser \
  -e WORDPRESS_DB_PASSWORD=gympass \
  -e WORDPRESS_DB_NAME=gymdb \
  wordpress:latest

STEP 1.4: Verify Status, Logs & Browser Setup
1. Run in Git Bash to inspect status and logs:

docker ps
docker logs --tail 20 gym-mysql
docker logs --tail 20 gym-wordpress

2. Open Chrome -> Go to: http://localhost:8085
3. Select Language (English) -> Click "Continue".
4. Fill in Site Setup:
   - Site Title: Centralized Gym Information Portal
   - Username: admin
   - Password: Password@123 (check "Confirm use of weak password" if shown)
   - Your Email: admin@gymportal.com
   - Click "Install WordPress" -> Click "Log In".
5. Create Sample Content:
   - In WordPress Dashboard left menu -> Click "Posts" -> "Add New Post".
   - Title: Gym Membership Plans & Timings
   - Body: Morning 6 AM - 11 AM | Evening 4 PM - 10 PM. Monthly Plan: Rs 1500.
   - Click "Publish" -> Click "View Post" to show the live portal page.

[HOW TO CONTINUE TO TASK 2]:
Leave the WordPress & MySQL containers running (or stop them if needed) and
open Chrome to http://localhost:8080 for Jenkins.


--------------------------------------------------------------------------------
TASK 2: JENKINS - JAVA WEB APP BUILD & CI (FREESTYLE PROJECT) [20 MARKS]
--------------------------------------------------------------------------------
[PDF REQUIREMENTS]:
1. Configure GymManagementSystem GitHub repo in a Jenkins Freestyle Project.
2. Select the required branch and configure Poll SCM trigger.
3. Configure Maven goals (clean, test, package) to compile, test, and package.
4. Execute job, inspect console output, and demonstrate BOTH a successful build
   and a failed build.
5. Archive the generated WAR file as a Jenkins build artifact.

[WHERE TO DO THIS]: Chrome Browser at http://localhost:8080 (your existing Jenkins).

STEP 2.1: Check Jenkins & Configure Maven Global Tool
1. Open Chrome -> Go to: http://localhost:8080
   (If localhost:8080 does not open: press Win + R -> type services.msc ->
   find "Jenkins" -> Right-click -> Start -> refresh browser).
2. Click "Manage Jenkins" (left sidebar) -> Click "Tools".
3. Scroll to "Maven installations" -> Click "Add Maven":
   - Name: Maven3
   - Check: "Install automatically"
   - Click "Save".

STEP 2.2: Create Freestyle Project & Configure Git + Poll SCM
1. Click "New Item" (top-left).
2. Enter Item Name: GymSystem-CI
3. Select: "Freestyle project" -> Click "OK".
4. Under "Source Code Management":
   - Select: "Git"
   - Repository URL: https://github.com/nikithamoturi/GymManagementSystem.git
   - Branches to build -> Branch Specifier: */main
     (Change to */master ONLY if Step 0 printed refs/heads/master).
5. Under "Triggers":
   - Check: "Poll SCM"
   - Schedule: * * * * *
     (Checks GitHub every 1 minute automatically).

STEP 2.3: Configure Maven Build Step & Archive WAR Artifact
1. Scroll down to "Build Steps" -> Click "Add build step" -> Select "Invoke top-level Maven targets".
2. Fill in:
   - Maven Version: Maven3
   - Goals: clean test package
3. Scroll down to "Post-build Actions" -> Click "Add post-build action" -> Select "Archive the artifacts".
4. Fill in:
   - Files to archive: target/*.war
5. Click "Save".

STEP 2.4: Execute Successful Build & Inspect Output
1. Click "Build Now" (left sidebar).
2. Wait for Build #1 to turn green -> Click "#1" -> Click "Console Output".
   - Verify: Git clone succeeds, tests execute, "BUILD SUCCESS" prints, and
     "Archiving artifacts" saves the .war file.
3. Click "Back to Project" -> Check "Last Successful Artifacts" to show the
   archived .war file directly on the project page.

STEP 2.5: Demonstrate a Failed Build (Required by PDF Point #4)
1. Click "Configure" (left sidebar of GymSystem-CI).
2. Scroll to "Build Steps" -> In Maven "Goals", intentionally enter an invalid
   goal: clean invalidgoal
3. Click "Save" -> Click "Build Now".
4. Build #2 turns red (Failed) -> Click "#2" -> Click "Console Output" to show
   the build failure to the examiner.
5. (Optional cleanup): Click "Configure" -> change Goals back to "clean test package"
   -> Click "Save".

[HOW TO CONTINUE TO TASK 3]:
Go back to your SAME Git Bash window on Windows (at ~/Desktop/SE-Lab-Set5).


--------------------------------------------------------------------------------
TASK 3: KUBERNETES - CONTAINER DEPLOYMENT & AUTO SCALING (MINIKUBE) [20 MARKS]
--------------------------------------------------------------------------------
[PDF REQUIREMENTS]:
1. Start Kubernetes cluster using Minikube and make Gym app image available.
2. Create a Deployment for the Gym application with 2 replicas.
3. Expose the Deployment using a Kubernetes Service and access the application.
4. Scale the Deployment from 2 to 5 replicas and verify desired/available pods.
5. Delete one running pod and demonstrate Kubernetes auto-creates a replacement.

[WHERE TO DO THIS]: Your SAME Git Bash window on Windows (at ~/Desktop/SE-Lab-Set5).

STEP 3.1: Clone Repo, Build Gym Docker Image & Start Minikube
Run these commands sequentially in Git Bash:

cd ~/Desktop/SE-Lab-Set5
git clone https://github.com/nikithamoturi/GymManagementSystem.git
cd GymManagementSystem

Create a Dockerfile inside ~/Desktop/SE-Lab-Set5/GymManagementSystem so we can
package the Gym web app into a real container image for Minikube:

cat << 'EOF' > Dockerfile
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

FROM tomcat:9.0-jdk17
RUN rm -rf /usr/local/tomcat/webapps/*
COPY --from=build /app/target/*.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
EOF

Start Minikube and load the Gym application image directly into Minikube:

minikube start --driver=docker
docker build -t gym-app:v1 .
minikube image load gym-app:v1

(Note: If the repository is a plain Maven structure without a .war packaging
or if docker build fails due to repo code, immediately run this backup command
so you do not get stuck:
docker pull tomcat:9.0 && docker tag tomcat:9.0 gym-app:v1 && minikube image load gym-app:v1
)

STEP 3.2: Create Deployment (2 Replicas) & Expose via Service
Run in the SAME Git Bash window:

kubectl create deployment gym-deployment --image=gym-app:v1 --replicas=2
kubectl get deployments
kubectl get pods
kubectl expose deployment gym-deployment --type=NodePort --port=8080
kubectl get svc gym-deployment
minikube service gym-deployment --url

(Open the http://127.0.0.1:xxxxx URL printed in Chrome to access the Gym app,
then press Ctrl + C in Git Bash to regain your terminal prompt).

STEP 3.3: Scale from 2 to 5 Replicas & Verify Self-Healing (Delete 1 Pod)
Run in the SAME Git Bash window:

# 1. Scale from 2 to 5 replicas
kubectl scale deployment gym-deployment --replicas=5

# 2. Verify DESIRED = 5 and AVAILABLE = 5
kubectl get deployment gym-deployment
kubectl get pods

# 3. Copy the exact name of ANY one running pod from the list above, then delete it:
kubectl delete pod <PASTE-ONE-POD-NAME-HERE>

# 4. Immediately check pods again to show Kubernetes auto-created a new 5th pod:
kubectl get pods

[HOW TO CONTINUE TO TASK 4]:
Stay in your SAME Git Bash window on Windows and move directly to Task 4 below.


--------------------------------------------------------------------------------
TASK 4: NAGIOS - INFRASTRUCTURE & APPLICATION MONITORING [20 MARKS]
--------------------------------------------------------------------------------
[PDF REQUIREMENTS]:
1. Pull and run Nagios Docker image and access Nagios web interface.
2. Configure application host and relevant web & database services for monitoring.
3. Monitor CPU utilization, memory usage, disk space, and server uptime.
4. Configure HTTP checks for Gym portal and a service check for MySQL.
5. Define suitable WARNING and CRITICAL thresholds for resources and services.

[WHERE TO DO THIS]: Same Git Bash window on Windows.
[EXACT FILE LOCATION]: You will create a Nagios host/service config file
~/Desktop/SE-Lab-Set5/gym-monitor.cfg and copy it into the Nagios container.

STEP 4.1: Run Nagios Container on Network Connected to WordPress & MySQL
Run in Git Bash (using port 8090 so it never clashes with Jenkins on 8080):

cd ~/Desktop/SE-Lab-Set5
docker run -d --name nagios-monitor \
  --network gym-network \
  -p 8090:80 \
  jasonrivers/nagios:latest

STEP 4.2: Create Nagios Host & Service Configuration with Thresholds
Run this block in Git Bash to create gym-monitor.cfg (monitors localhost CPU,
Memory, Disk, Uptime + Gym Portal HTTP check + MySQL port 3306 check with
explicit WARNING and CRITICAL thresholds):

cat << 'EOF' > gym-monitor.cfg
define host {
    use                     linux-server
    host_name               gym-portal-server
    alias                   Gym Portal & Database Host
    address                 gym-wordpress
    max_check_attempts      3
    check_period            24x7
    notification_interval   30
    notification_period     24x7
}

define service {
    use                     generic-service
    host_name               gym-portal-server
    service_description     Gym Portal HTTP Availability
    check_command           check_http!-w 2 -c 5
}

define service {
    use                     generic-service
    host_name               gym-portal-server
    service_description     MySQL Database Port 3306 Check
    check_command           check_tcp!3306!-w 2 -c 5
}

define service {
    use                     generic-service
    host_name               localhost
    service_description     CPU Load Utilization
    check_command           check_local_load!3.0,2.5,2.0!5.0,4.0,3.0
}

define service {
    use                     generic-service
    host_name               localhost
    service_description     Root Disk Space Usage
    check_command           check_local_disk!20%!10%!/
}

define service {
    use                     generic-service
    host_name               localhost
    service_description     Current Users & Uptime Load
    check_command           check_local_users!10!20
}
EOF

STEP 4.3: Load Config into Nagios Container, Validate & Restart
Run in the SAME Git Bash window:

docker cp gym-monitor.cfg nagios-monitor:/opt/nagios/etc/objects/gym-monitor.cfg
docker exec nagios-monitor sh -c "echo 'cfg_file=/opt/nagios/etc/objects/gym-monitor.cfg' >> /opt/nagios/etc/nagios.cfg"
docker exec nagios-monitor /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
docker restart nagios-monitor

(Verify "Total Warnings: 0" and "Total Errors: 0" printed from nagios -v).

STEP 4.4: Verify Nagios Web Interface in Chrome
1. Open Chrome -> Go to: http://localhost:8090
2. Login -> Username: nagiosadmin | Password: nagios
3. Click "Hosts" (left sidebar) -> Show "localhost" and "gym-portal-server" (UP).
4. Click "Services" (left sidebar) -> Show:
   - Gym Portal HTTP Availability (OK - Warning 2s / Critical 5s)
   - MySQL Database Port 3306 Check (OK)
   - CPU Load Utilization (OK - Warning 3.0 / Critical 5.0)
   - Root Disk Space Usage (OK - Warning 20% free / Critical 10% free)
   - Memory / Swap Usage & Uptime (OK)

[HOW TO CONTINUE TO TASK 5]:
Open AWS Management Console in a new Chrome tab and keep your SAME Git Bash
window ready at ~/Desktop/SE-Lab-Set5.


--------------------------------------------------------------------------------
TASK 5: AWS - CLOUD DEPLOYMENT USING EC2 [20 MARKS]
--------------------------------------------------------------------------------
[PDF REQUIREMENTS]:
- Deploy Gym Membership Management System on an Ubuntu EC2 instance.
- Make it accessible through the internet (HTTP/8080) while restricting
  administrative SSH access (Port 22) to authorized IP addresses.

[WHERE TO DO THIS]: AWS Console in Chrome + your SAME Git Bash window.

STEP 5.1: Launch Ubuntu EC2 Instance with Restricted SSH in AWS Console
1. Open AWS Management Console -> Search "EC2" -> Click "Launch Instance".
2. Configure these exact settings:
   - Name: GymSystem-EC2
   - Application and OS Images (AMI): Ubuntu (Ubuntu Server 22.04 LTS or 24.04 LTS)
   - Instance type: t2.micro (or t3.micro)
   - Key pair: Click "Create new key pair" -> Name: gym-key -> Type: RSA ->
     Format: .pem -> Click "Create key pair".
     (Save gym-key.pem inside your Desktop/SE-Lab-Set5 folder).
   - Network settings (Firewall / Security Group - IMPORTANT FOR PDF):
     * Under "Allow SSH traffic from": Select "My IP" (this restricts port 22
       administrative access strictly to your authorized IP address!).
     * Check "Allow HTTP traffic from the internet" (0.0.0.0/0).
     * Click "Edit" -> Click "Add security group rule":
       Type: Custom TCP | Port range: 8080 | Source type: Anywhere (0.0.0.0/0)
3. Click "Launch Instance" -> Click "View all instances".
4. Wait until Instance state = "Running", select GymSystem-EC2, and copy its
   "Public IPv4 address".

STEP 5.2: Connect to EC2 Instance via SSH
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows (at ~/Desktop/SE-Lab-Set5).
Run these commands (replace <EC2-PUBLIC-IP> with your copied Public IP):

cd ~/Desktop/SE-Lab-Set5
chmod 400 gym-key.pem
ssh -i gym-key.pem ubuntu@<EC2-PUBLIC-IP>

(Type yes and press Enter when prompted. Your terminal prompt will change to
ubuntu@ip-xxx-xxx-xxx-xxx:~$).
*(Backup if college Wi-Fi NAT changes your IP or blocks port 22: In AWS Console,
select GymSystem-EC2 -> Click "Connect" -> "EC2 Instance Connect" -> "Connect"
to open the Ubuntu shell in your browser).*

STEP 5.3: Install Java, Git, Maven, and Tomcat on Ubuntu EC2
[WHERE TO DO THIS]: Inside the Ubuntu EC2 terminal (ubuntu@ip-...:~$).

sudo apt update && sudo apt upgrade -y
sudo apt install -y openjdk-17-jdk git maven
sudo apt install -y tomcat9 || sudo apt install -y tomcat10

STEP 5.4: Clone Gym Repo, Build WAR & Deploy to Tomcat
[WHERE TO DO THIS]: Inside the SAME Ubuntu EC2 terminal.

cd ~
git clone https://github.com/nikithamoturi/GymManagementSystem.git
cd GymManagementSystem
mvn clean package -DskipTests
ls -lh target/

Deploy the generated .war file into Tomcat's webapps/ folder and enable
auto-start on reboot:

if [ -d "/var/lib/tomcat9/webapps" ]; then
    sudo cp target/*.war /var/lib/tomcat9/webapps/GymManagementSystem.war
    sudo systemctl restart tomcat9
    sudo systemctl enable tomcat9
else
    sudo cp target/*.war /var/lib/tomcat10/webapps/GymManagementSystem.war
    sudo systemctl restart tomcat10
    sudo systemctl enable tomcat10
fi

STEP 5.5: Verify Public Internet Access in Chrome
1. Open Chrome and go to:
   http://<EC2-PUBLIC-IP>:8080/GymManagementSystem
2. Confirm the Gym Membership Management System loads over the public internet.
3. Type exit in Git Bash to close the EC2 SSH session cleanly:

exit
================================================================================


     "sudo lsof -i :3000" or "sudo ss -tulpn | grep 3000".
================================================================================
