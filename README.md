


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



     "sudo lsof -i :3000" or "sudo ss -tulpn | grep 3000".
================================================================================
