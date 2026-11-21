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
    Go straight to Task A.
- If Chrome says "This site can't be reached / localhost refused to connect":
  - Press Win + R on your keyboard -> type: services.msc -> press Enter.
  - Scroll down to "Jenkins" -> Right-click "Jenkins" -> Click "Start".
  - Refresh http://localhost:8080 in Chrome.
    *(Alternative if installed via jenkins.war instead of Windows Service: open
    Command Prompt in the folder containing jenkins.war and run:
    java -jar jenkins.war --httpPort=8080)*


--------------------------------------------------------------------------------
TASK A: DOCKER - MULTI-CONTAINER APPLICATION [20 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows + Chrome.
[EXACT FOLDER & FILES]: Create ~/Desktop/SE-Lab-Exam/metro-docker and write
all files inside that exact same folder:
1. ~/Desktop/SE-Lab-Exam/metro-docker/metro-reservation/Dockerfile
2. ~/Desktop/SE-Lab-Exam/metro-docker/metro-reservation/docker-compose.yml

STEP A.1: Clone Repo, Build WAR & Write Dockerfile + docker-compose.yml
Copy and paste this entire block into Git Bash:

cd ~/Desktop/SE-Lab-Exam
mkdir -p metro-docker
cd metro-docker
git clone https://github.com/deepthisagar7/metro-reservation.git
cd metro-reservation
mvn clean package
ls -lh target/

cat << 'EOF' > Dockerfile
FROM tomcat:9.0-jdk17
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/*.war /usr/local/tomcat/webapps/metro-reservation.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
EOF

cat << 'EOF' > docker-compose.yml
version: "3.8"

services:
  mysql-db:
    image: mysql:8.0
    container_name: metro-mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: metro_reservation_db
      MYSQL_USER: metrouser
      MYSQL_PASSWORD: micropassword
    ports:
      - "3306:3306"
    volumes:
      - metro-mysql-data:/var/lib/mysql
    networks:
      - metro-net
    restart: unless-stopped

  metro-app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: metro-web
    ports:
      - "8081:8080"
    environment:
      DB_HOST: mysql-db
      DB_PORT: "3306"
      DB_NAME: metro_reservation_db
      DB_USER: metrouser
      DB_PASSWORD: micropassword
    depends_on:
      - mysql-db
    networks:
      - metro-net
    restart: unless-stopped

networks:
  metro-net:
    driver: bridge

volumes:
  metro-mysql-data:
EOF

ls -la

(Note: Host port 8081 is mapped to container port 8080 so it does not clash
with local Jenkins on 8080).

STEP A.2: Pull Images, Start Containers & Verify in Chrome
Run these commands in Git Bash:

docker compose pull
docker compose up -d --build
docker compose ps
docker network ls | grep metro
docker volume ls | grep metro

1. Open Chrome and go to: http://localhost:8081/metro-reservation
2. Confirm the Metro Reservation web app loads in the browser.

SCENARIO-BASED QUESTIONS & ANSWERS (TASK A):
1. What happens if the MySQL container stops while passengers are accessing the Metro Reservation application?
   -> Static UI pages may briefly load from cache, but any database-dependent
      operation (login, train search, seat booking, reservation lookup) fails
      with connection errors or HTTP 500 errors until MySQL is restarted.
2. How can you preserve passenger reservation data even after deleting the MySQL container?
   -> Mount a Docker named volume (e.g., metro-mysql-data:/var/lib/mysql) or host
      bind mount. Volumes persist independently of the container lifecycle, so
      deleting the MySQL container keeps all data intact for future containers.
3. Why should the Metro Reservation application and MySQL database run in separate containers?
   -> It provides isolation, independent scaling, easier restarts/upgrades
      without impacting the database, fault containment, and separate resource
      and security configurations.
4. How can you prevent unauthorized access to passenger and reservation information?
   -> Enforce application authentication/RBAC, use HTTPS/TLS in transit, avoid
      exposing MySQL port 3306 publicly, keep MySQL on an internal bridge network,
      and use strong, least-privilege database credentials.
5. How would you troubleshoot the application if it cannot connect to the MySQL database?
   -> Check container status (`docker compose ps`), inspect logs (`docker logs
      metro-mysql` and `docker logs metro-web`), verify both containers are on
      the same network (`metro-net`), check DB host/port/user/password/DB name
      configs, and test port 3306 reachability from inside `metro-web`.


--------------------------------------------------------------------------------
TASK B: JENKINS - FREESTYLE PROJECT [20 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Chrome Browser at http://localhost:8080.
[FILES NEEDED ON DISK]: None. Everything is configured inside the Jenkins web UI.

STEP B.1: Configure JDK & Maven in Jenkins Tools
1. In Chrome at http://localhost:8080, log into Jenkins.
2. Click "Manage Jenkins" (left menu) -> Click "Tools".
3. Under "JDK installations", verify a JDK is configured (or add one).
4. Scroll down to "Maven installations" -> Click "Add Maven".
5. Enter:
   - Name: Maven3
   - Check the box: "Install automatically"
6. Click "Save" at the bottom.

STEP B.2: Create Freestyle Project & Configure Git + Maven Build
1. Click "New Item" (top-left).
2. Enter Item Name: Metro-Reservation-Freestyle
3. Click "Freestyle project" -> Click "OK".
4. Under "Source Code Management":
   - Select: Git
   - Repository URL: https://github.com/deepthisagar7/metro-reservation.git
   - Branch Specifier: */main (or */master if default branch is master)
5. Under "Build Triggers":
   - Check: "GitHub hook trigger for GITScm polling"
   - Check: "Poll SCM" and enter: * * * * *
6. Under "Build Steps":
   - Click "Add build step" -> Select "Invoke top-level Maven targets"
   - Maven Version: Maven3
   - Goals: clean package
   *(Alternative: "Execute Windows batch command" -> mvn clean package)*

STEP B.3: Execute & Verify Output
1. Click "Save" -> Click "Build Now" (left menu).
2. Click "#1" under Build History -> Click "Console Output".
3. Verify: "BUILD SUCCESS" appears and the WAR file (e.g.,
   metro-reservation.war) is generated inside target/.

SCENARIO-BASED QUESTIONS & ANSWERS (TASK B):
1. What happens if a developer introduces compilation errors into the Metro Reservation application?
   -> Maven fails during the compile phase, aborts the build before packaging
      a WAR file, and Jenkins marks the build as FAILURE while logging the
      exact compiler errors in Console Output.
2. How can Jenkins detect changes made to the GitHub repository?
   -> Using Poll SCM (Jenkins checks GitHub periodically on a cron schedule like
      * * * * *) or a GitHub Webhook (GitHub notifies Jenkins immediately via
      HTTP POST on every push).
3. What is the difference between successfully generating the WAR file and deploying the application on Tomcat?
   -> Generating the WAR file compiles code and bundles the archive in target/.
      Deploying copies that WAR file into Tomcat's webapps/ directory so Tomcat
      extracts and serves the application live over HTTP.
4. Why is Maven useful for managing dependencies and building the Metro Reservation application?
   -> Maven reads pom.xml to automatically download and resolve required JAR
      dependencies, enforces a standard build lifecycle (compile, test, package),
      and ensures consistent builds across all machines.
5. How would you troubleshoot a Jenkins build failure caused by an incorrect pom.xml file?
   -> Open the failed build's Console Output in Jenkins, read the XML syntax/
      dependency/plugin error message, verify <packaging>war</packaging> and
      dependency versions in pom.xml, fix and push the file, and rebuild.


--------------------------------------------------------------------------------
TASK C: SYSTEM MONITORING USING NAGIOS [20 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows + Chrome.

STEP C.1: Pull and Run Nagios Docker Image
(Jenkins is using port 8080 locally, so Nagios is mapped to host port 8090):

docker run --name nagios4 -p 8090:80 -d jasonrivers/nagios:latest
docker ps

STEP C.2: Configure Host & HTTP Service Monitoring YAML Reference
(Save a reference YAML or inspect Nagios config inside the container to monitor
CPU load, memory, disk space, uptime, and HTTP availability on port 8081):

cd ~/Desktop/SE-Lab-Exam/metro-docker
cat << 'EOF' > nagios-monitor-config.yml
nagios_monitoring:
  host:
    host_name: metro-reservation-server
    alias: Metro Reservation App Server
    address: host.docker.internal
    check_command: check-host-alive
  services:
    - service_description: HTTP Metro Reservation
      check_command: check_http!-I host.docker.internal -p 8081 -u /metro-reservation
    - service_description: Current CPU Load
      check_command: check_local_load!5.0,4.0,3.0!10.0,6.0,4.0
    - service_description: Root Partition Disk Space
      check_command: check_local_disk!20%!10%!/
    - service_description: Memory Usage
      check_command: check_mem!-w 80 -c 90
    - service_description: System Uptime
      check_command: check_uptime
EOF

STEP C.3: Verify Nagios Dashboard in Chrome
1. Open Chrome -> Go to: http://localhost:8090
2. Log in with:
   - Username: nagiosadmin
   - Password: nagios
3. Click "Hosts" and "Services" on the left sidebar to inspect host status,
   CPU load, disk space, uptime, and HTTP service health.

SCENARIO-BASED QUESTIONS & ANSWERS (TASK C):
1. How can Nagios identify whether the Metro Reservation application server is running or down?
   -> Nagios runs periodic host checks (such as ICMP check_ping or TCP checks).
      If the server stops responding across configured retries, Nagios flags
      the host status as DOWN.
2. What is the difference between WARNING and CRITICAL alerts?
   -> WARNING means a metric crossed an early threshold (e.g., high CPU or slow
      response) requiring attention before failure. CRITICAL means a severe
      threshold breach or outright outage (e.g., service stopped, HTTP 500, or
      disk full) requiring immediate action.
3. What happens if CPU usage reaches 100% during peak ticket reservations?
   -> Request queues build up, page load times spike, threads get starved, and
      passengers experience timeouts or failed ticket bookings.
4. How can Nagios identify whether the Metro Reservation application is unavailable even when the server is running?
   -> Host ping only checks OS/network reachability, whereas an HTTP service check
      (check_http on port 8080/8081 and path /metro-reservation) tests the web
      app endpoint directly and triggers a CRITICAL alert if it fails.
5. How can an administrator identify whether downtime is caused by Tomcat, MySQL or the server?
   -> Compare Nagios check results: if Host check fails -> Server/Network issue;
      if Host is UP but check_http fails -> Tomcat/App issue; if Tomcat is UP
      but DB queries/health checks fail -> MySQL issue.


--------------------------------------------------------------------------------
TASK D: CONTAINER SCALING USING KUBERNETES [20 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows.

STEP D.1: Start Minikube, Build Image & Write Kubernetes YAML
Copy and paste this entire block into Git Bash:

cd ~/Desktop/SE-Lab-Exam/metro-docker/metro-reservation
minikube start --driver=docker
kubectl get nodes

docker build -t metro-reservation:latest .
minikube image load metro-reservation:latest

cat << 'EOF' > metro-k8s.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metro-reservation-app
  labels:
    app: metro-reservation
spec:
  replicas: 2
  selector:
    matchLabels:
      app: metro-reservation
  template:
    metadata:
      labels:
        app: metro-reservation
    spec:
      containers:
        - name: metro-reservation
          image: metro-reservation:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: metro-reservation-svc
spec:
  type: NodePort
  selector:
    app: metro-reservation
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
EOF

STEP D.2: Deploy, Scale Replicas & Demonstrate Pod Auto-Recreation
Run these commands in Git Bash:

kubectl apply -f metro-k8s.yaml
kubectl get pods
kubectl get svc metro-reservation-svc

kubectl scale deployment metro-reservation-app --replicas=4
kubectl get pods -l app=metro-reservation

POD_NAME=$(kubectl get pods -l app=metro-reservation -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD_NAME
kubectl get pods -l app=metro-reservation

(Verify that Kubernetes immediately creates a new pod to maintain 4 replicas.
Optional: run `minikube service metro-reservation-svc --url` to view in Chrome,
then press Ctrl + C to return to prompt).

SCENARIO-BASED QUESTIONS & ANSWERS (TASK D):
1. What happens if one Metro Reservation application pod crashes during ticket booking?
   -> Only requests currently handled by that crashed pod are interrupted;
      other healthy pods keep serving traffic, and Kubernetes automatically
      starts a replacement pod.
2. How does Kubernetes maintain the desired number of application replicas?
   -> The ReplicaSet controller runs a continuous reconciliation loop comparing
      actual running pods against spec.replicas, creating new pods if any fail
      or terminating excess pods if above target.
3. What is the difference between scaling up and scaling down?
   -> Scaling up increases pod replicas to handle heavier user traffic and load;
      scaling down reduces pod replicas during low-traffic periods to save CPU,
      memory, and infrastructure cost.
4. How can increasing the number of application replicas help during peak reservation hours?
   -> Multiple replicas spread concurrent passenger requests across multiple
      containers, preventing CPU/memory saturation on a single instance and
      reducing response latency.
5. How can Kubernetes distribute incoming requests among multiple application pods?
   -> A Kubernetes Service selects all ready pods matching its label selector
      and load-balances incoming traffic across their pod IPs automatically.


--------------------------------------------------------------------------------
TASK E: AWS DEPLOYMENT USING EC2 [20 MARKS]
--------------------------------------------------------------------------------
[WHERE TO DO THIS]: AWS Console in Chrome + your SAME Git Bash window.

STEP E.1: Provision Ubuntu EC2 Instance in AWS Console
1. Open AWS Management Console -> Search "EC2" -> Click "Launch Instance".
2. Configure these exact settings:
   - Name: MetroReservation-EC2
   - Application and OS Images (AMI): Ubuntu (Ubuntu Server 22.04 LTS or 24.04 LTS)
   - Instance type: t2.micro (or t3.micro)
   - Key pair: Click "Create new key pair" -> Name: metro-key -> Type: RSA ->
     Format: .pem -> Click "Create key pair".
     (Save metro-key.pem inside your Desktop/SE-Lab-Exam folder).
   - Network settings (Firewall / Security Group):
     - Check "Allow SSH traffic from" (0.0.0.0/0)
     - Check "Allow HTTP traffic from the internet"
     - Click "Edit" -> Click "Add security group rule":
       Type: Custom TCP | Port range: 8080 | Source type: Anywhere (0.0.0.0/0)
3. Click "Launch Instance" -> Click "View all instances".
4. Wait until Instance state = "Running", select MetroReservation-EC2, and copy
   its "Public IPv4 address".

STEP E.2: Connect to EC2 Instance via SSH
[WHERE TO DO THIS]: Your SAME Git Bash window on Windows (at ~/Desktop/SE-Lab-Exam).
Run these commands (replace <EC2-PUBLIC-IP> with your copied Public IP):

cd ~/Desktop/SE-Lab-Exam
chmod 400 metro-key.pem
ssh -i metro-key.pem ubuntu@<EC2-PUBLIC-IP>

(Type yes and press Enter when prompted. Backup if port 22 is blocked: select
instance in AWS Console -> Click "Connect" -> "EC2 Instance Connect").

STEP E.3: Install Java, Git, Maven, Tomcat & Deploy WAR on EC2
[WHERE TO DO THIS]: Inside the Ubuntu EC2 terminal (ubuntu@ip-...:~$).

sudo apt update && sudo apt upgrade -y
sudo apt install -y openjdk-17-jdk git maven
sudo apt install -y tomcat9 || sudo apt install -y tomcat10

cd ~
git clone https://github.com/deepthisagar7/metro-reservation.git
cd metro-reservation
mvn clean package
ls -lh target/

if [ -d "/var/lib/tomcat9/webapps" ]; then
    sudo cp target/*.war /var/lib/tomcat9/webapps/metro-reservation.war
    sudo systemctl restart tomcat9
    sudo systemctl enable tomcat9
else
    sudo cp target/*.war /var/lib/tomcat10/webapps/metro-reservation.war
    sudo systemctl restart tomcat10
    sudo systemctl enable tomcat10
fi

STEP E.4: Verify Public Access in Chrome
1. Open Chrome and go to: http://<EC2-PUBLIC-IP>:8080/metro-reservation
2. Confirm the Metro Reservation web app loads over Public IP.
3. Type `exit` in the EC2 terminal to return to your local Windows Git Bash.

SCENARIO-BASED QUESTIONS & ANSWERS (TASK E):
1. What could be the reasons if passengers cannot access the Metro Reservation application using the EC2 public IP address?
   -> Port 8080 is closed in the EC2 Security Group or OS firewall, Tomcat is
      stopped or failed to deploy the WAR file, the URL path/port is wrong, or
      the instance's public IP changed after a stop/start without an Elastic IP.
2. Why should HTTP and application server ports be configured in the EC2 security group?
   -> EC2 Security Groups act as virtual firewalls blocking all inbound traffic
      by default; opening ports 80 and 8080 explicitly allows external user
      browsers to reach the web/application server.
3. What happens if Tomcat stops while the EC2 instance is still running?
   -> The Linux VM remains running and reachable via SSH/ping, but the web
      application stops serving pages and users get connection refused errors
      until Tomcat is restarted.
4. How can you make the Metro Reservation application available if the EC2 instance fails?
   -> Deploy multiple EC2 instances across multiple Availability Zones behind
      an Application Load Balancer (ALB) with an Auto Scaling Group and external
      database (RDS) so traffic automatically fails over to healthy instances.
5. What are the advantages of deploying the Metro Reservation application on AWS instead of a local server?
   -> AWS provides global public accessibility, elastic on-demand scaling, high
      availability across Availability Zones, pay-as-you-go pricing without
      physical hardware upkeep, and managed networking/security tools.
