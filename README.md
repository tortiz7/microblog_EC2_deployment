# Microblog Deployment on Self-Provisioned Infrastructure with Monitoring

---
### Purpose

The purpose of this project is to deploy a Microblog Flask application on self-built infrastructure using AWS EC2 instances. Through this workload, we explored various concepts related to cloud infrastructure, Continuous Integration/Continuous Deployment (CI/CD), and application monitoring. By configuring and managing our own servers, we strengthened our skills in system administration, security group management, and application deployment.

This project emphasizes hands-on experience with tools such as Jenkins for automation, Prometheus for monitoring, and Grafana for visualization. Additionally, we learned to troubleshoot and optimize our setup, ensuring that our application operates efficiently and is resilient against potential issues. Overall, this workload provides a comprehensive understanding of how to create and manage a robust application infrastructure from the ground up.

---
## STEPS

**Create EC2 Instances and AWS Access Keys**
- **Why**: To launch the Microblog Flask application, monitor the server that is running it, and gain acces to these two servers, we will need to create an AWS Access Key and two EC2's. AWS Access Keys are necessary for programmatic access to AWS services. The key we generate will be used to securely SSH into the EC2's.
  
- **How:**
- **Create the AWS Access Key:**
    1. Navigate to the AWS service: IAM (search for this in the AWS console)
    2. Click on "Users" on the left side navigation panel
    3. Click on your User Name
    4. Underneath the "Summary" section, click on the "Security credentials" tab
    5. Scroll down to "Access keys" and click on "Create access key"
    6. Select the appropriate "use case", and then click "Next" and then "Create access key"

The Access and Secret Access keys are needed for future steps, so safe storage of them is vital to a successful automated CI/CD pipeline. **Never** share your access keys, as a bad actor can get a hold of them and use the keys to access your server, wreaking havoc, compromising data integrity and potentially stealing sensitive information.

  - **Create the two EC2's and their Security Groups:**
    1. Navigate to the EC2 page in AWS and click "Launch Instance"
    2. For the Jenkins server, name it "Jenkins" and select "Ubuntu" as the OS Image. For the EC2 that will 
       have Prometheus and Grafana installed, name it "Monitoring".
    3. For Jenkins, select a t3.medium as the instance type (lest your OWASP Scan stage for Jenkins build will 
       crash). For the Monitoring EC2, a t3.micro will suffice.
    4. Select the key pair you just created as your method of SSH'ing into the EC2's. 
    5. Create two Security Groups that allow inbound traffic to the services and applications the EC2's will 
       need:
    	i. The Jenkins Security Group will need these ports open: 22 (for SSH access) 80 (for NGNIX to forward 
           to 
           Gunicorn), 8080 (for Jenkins), and 9100 (for the Node Exporter, whose use I will explain later)
	ii. The Monitoring EC2 Security group requires additional ports: 22 (for SSH access), 9100 (for the 
            node exporter), 3000 (for Grafana) and 9090 (for Prometheus)
               
---
### Install Jenkins
- **Why**: Jenkins automates the build and deployment pipeline. It pulls code from GitHub, tests it, and handles deployment once the Jenkinsfile is configured to do so. We've previously used Jenkins for CI/CD implementation for previous workloads - I've built upon foundation, this time adding pytest for the testing phase and OWASP scanning to ensure our dependencies are ready for deployment. 
  
- **How**: I created and ran the below script to install Jenkins, and it's language prerequisite Java (using Java 11 for this deployment). To save time, the script first updates and upgrades all packages included in the EC2, ensuring they are up-to-date and secure. The script also installs Python3.9 - the langauge our flask app relies on - and all the dependencies necessary for our application (Python3.9 venv and Python3.9-pip), as well as Nginix, a reverse-proxy server.

``` bash
#!/bin/bash
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y openjdk-11-jdk
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 5BA31D57EF5975CA
sudo apt update -y
sudo apt install -y jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update -y
sudo apt install -y python3.9 python3.9-venv python3-pip
sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
echo "Jenkins initial password:"
sudo cat /var/lib/jenkins/secrets/initialAdminPassword!
```

---
### Configure the NGINX Location block
- **Why:** NGINX is a reverse proxy server between the client browser and our Microblog application. It is a gatekeeper, managing incoming traffic to assist with load balancing in the case of scalability, strengthening security by validating SSL/TLS certificates for incoming HTTPS requests, and increasing performance by caching certain static content (images, JavaScript files) and forwarding all dynamic content to Gunicorn (on Port 5000, only exposed locally, and not to the internet).

- **How:** We nano into the NGINX Configuration file at this location: `/etc/nginx/sites-enabled/default` and add this to the Location block:
  
```bash
  location / {
proxy_pass http://127.0.0.1:5000;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}  
```

---
### Configure and Run the Microblog App on the EC2
- **Why:** Before we can automate the deployment of the Microblog application via Jenkins, we want to ensure we have every component we need to run it. To do so, we will clone the source code repository from GitHub to our EC2's, and then create a venv in that directory, install all dependencies, set the environment variable, compile and migrate the databases, and then run Gunicorn to serve the app.

 - **How:**
 -  ```bash
     git clone https://github.com/tortiz7/microblog_EC2_deployment.git
     cd microblog_EC2_deployment
     python3.9 -m venv venv
     source venv/bin/activate
     ```
   - Install dependencies:
     ```bash
     pip install -r requirements.txt
     pip install gunicorn pymysql cryptography
     ```
   - The next three commands were new to me, and do the following (you'd run them in the terminal as commands as they are written below):
   - **`FLASK_APP=microblog.py`**: Sets the Flask application context, directing the Flask CLI to use `microblog.py` as the entry point for the application. This is essential for running and managing the app.

   - **`flask translate compile`**: Compiles translation files, converting `.po` files into `.mo` files for Flask-Babel, enabling multi-language support. This command ensures that the latest translations are incorporated into the application.

   - **`flask db upgrade`**: Applies database migrations to update the schema according to the current models in the application. This command is vital for maintaining the integrity of the database and ensuring that all changes to the data structure are reflected accurately.
  
   - Run Gunicorn to serve the app:
     ```bash
     gunicorn -b :5000 -w 4 microblog:app
     ```
    
---
### Create Gunicorn Daemon

- **Why**: We created a Gunicorn daemon to ensure the Microblog app runs as a service and automatically starts on boot. This helps manage the app's lifecycle, ensuring that Gunicorn starts, stops, and restarts as needed without manual intervention. More on this in the **Issues/Troubleshooting** section below.
  
- **Where**: The Gunicorn service file was created in the `/etc/systemd/system/` directory on the Jenkins EC2 instance. This is where system-level services are managed on Linux systems.

- **Service File Contents**: Below is the configuration we used for the Gunicorn daemon:
  
```bash
Unit]
Description=Gunicorn instance to serve my application
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/WL3
Environment="FLASK_APP=microblog.py"
ExecStart=/home/ubuntu/WL3/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:5000 microblog:app
Restart=always

[Install]
WantedBy=multi-user.target





```

---
### Create `test_app.py` Script for Pytest
### Create `test_app.py` Script for Pytest

- **Why:** We created the `test_app.py` script for Pytest to automate testing of key functionality in the Microblog app, specifically ensuring that critical components such as the homepage and login functionality work as expected. This is vital for continuous integration (CI) pipelines, as automated tests help identify issues early in the build process, preventing bugs from reaching production and improving the overall quality of the application.

- **How:** The `test_app.py` script performs multiple tests:

    - **Homepage test:** This test ensures the homepage loads correctly and returns the expected content. The `client.get('/', follow_redirects=True)` checks if the root URL is accessible and returns an HTTP 200 response. The inclusion of `b'Microblog'` in the response ensures that the page contains expected content, confirming that the homepage is functional.

    - **Login page test:** This test verifies that the `/auth/login` page, responsible for user authentication, is accessible and functional. Like the homepage test, it ensures the login page is correctly rendered and that its core elements (like the page title) are loaded.

    - **404 test:** This test ensures the application appropriately handles non-existent routes. When an invalid URL is accessed, the app should return a 404 error. Testing for this behavior ensures the app doesn't crash and provides proper feedback to users when they encounter broken or incorrect links.

```python
import pytest
from microblog import create_app

import pytest
from microblog import create_app

@pytest.fixture
def client():
  app = create_app()
  app.config['TESTING'] = True
  with app.test_client() as client:
    with app.app_context():
      yield client

def test_home_page(client):
  response = client.get('/', follow_redirects=True)
  assert response.status_code == 200
  assert b'Microblog' in response.data

def test_login_page(client):
  response = client.get('/auth/login')
  assert response.status_code == 200
  assert b'Microblog' in response.data

def test_404_page(client):
  response = client.get('/nonexistent')
  assert response.status_code == 404
```


---
### Set the 'Jenkins' User as a Sudoer
- **Why:** The Jenkinsfile Deploy stage requires Jenkins to run a command with sudo, which requires the Jenkins user to have Sudoer permissions. I don't want to ruin all the fun - you'll see what that command is below, in the "Issues/Troubleshooting" section.

- **How:** To add the Jenkins user as a Sudoer, you first must type the `sudo visudo` command in the Jenkins EC2 terminal to edit the Sudoers file. Then, underneath the Root user entry in the Sudoers file, add the following:
```bash
jenkins ALL=(ALL) NOPASSWD: ALL
```
The above command will allow the Jenkins user to run any sudo command without the need of a password. Not the most secure solution, but since the Jenkinsfile will only have the Jenkins user running one Sudo command, it is fine for our purposes. 

---
### Configure Jenkins Pipeline
- **Why**: The Jenkins pipeline builds the application, tests it's logic to ensure no errors, and deploys the Microblog application to the web for us. For this workload, we also implemented stages to Clean the deployment server to ensure there was no conflicts with stale Gunicorn proccesses, and the OWASP Dependency check, which scans our third-party libraries and dependencies (such as our Python packages) against the National Vulnerability Database, generating a report that informs us what packages might be at risk and allowing us update or replace them before an issue can arise
  
- **How**: The Jenkinsfile was written and configured in these stages:
  
     - **Build Stage**: Created virtual environment, installed dependencies, ran migrations, and compiled translations.
     - **Test Stage**: Configured Pytest to run unit tests for the homepage and login routes.
     - **Clean Stage**: Stopped running Gunicorn instances before each new deploy.
     - **OWASP Dependency Check**: Added the OWASP plugin to check for known vulnerabilities.
     - **Deploy Stage**: Deployed the app by running Gunicorn as a daemon process.

---
### Install Prometheus and Grafana on the Monitoring EC2

- **Why**: Prometheus and Grafana are critical for monitoring the health and performance of servers. Prometheus scrapes system metrics from the Jenkins EC2, while Grafana visualizes those metrics for easier analysis. Monitoring helps identify performance bottlenecks, resource usage trends, and potential system failures before they impact the app.

- **How**: 
    1. **Install Prometheus**:
       ```bash
       sudo apt update
       sudo apt install -y wget tar
       wget https://github.com/prometheus/prometheus/releases/download/v2.36.0/prometheus-2.36.0.linux-amd64.tar.gz
       tar -xvzf prometheus-2.36.0.linux-amd64.tar.gz
       sudo mv prometheus-2.36.0.linux-amd64 /usr/local/prometheus
       ```
    2. **Create a service daemon for Prometheus**:
       - To ensure Prometheus starts automatically:
         ```bash
         sudo nano /etc/systemd/system/prometheus.service
         ```
         Add the following to the file:
         ```bash
         [Unit]
         Description=Prometheus Monitoring
         After=network.target

         [Service]
         User=prometheus
         ExecStart=/home/ubuntu/prometheus-2.54.1.linux-amd64/prometheus \
         --config.file=/home/ubuntu/prometheus-2.54.1.linux-amd64/prometheus.yml \
         --storage.tsdb.path=/home/ubuntu/prometheus-2.54.1.linux-amd64/data
         Restart=always

         [Install]
         WantedBy=multi-user.target
         ```
         - Start and enable the service:
           ```bash
           sudo systemctl daemon-reload
           sudo systemctl start prometheus
           sudo systemctl enable prometheus
           ```

    3. **Install Grafana**:
       - Add the Grafana APT repository:
         ```bash
         sudo apt install -y software-properties-common
         sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
         wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add
         sudo apt update
         sudo apt install -y grafana
         ```
       - Start and enable Grafana:
         ```bash
         sudo systemctl start grafana-server
         sudo systemctl enable grafana-server
	 ```
  
---
### Install Node Exporter on the Jenkins EC2

- **Why**: Node Exporter is a Prometheus exporter that collects system-level metrics such as CPU, memory, and disk usage from the Jenkins EC2. This is essential for monitoring system health and resource usage on the Jenkins server.

- **How**: 
    1. **Install Node Exporter**:
       ```bash
       wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
       tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
       sudo mv node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/bin/
       ```
    2. **Create a service daemon for Node Exporter**:
       ```bash
       sudo nano /etc/systemd/system/node_exporter.service
       ```
       Add the following to the file:
       ```bash
       [Unit]
       Description=Node Exporter
       After=network.target

       [Service]
       User=node_exporter
       ExecStart=/usr/local/bin/node_exporter

       [Install]
       WantedBy=multi-user.target
       ```
       - Start and enable the Node Exporter service:
         ```bash
         sudo systemctl daemon-reload
         sudo systemctl start node_exporter
         sudo systemctl enable node_exporter
         ```

---
### Configure Prometheus to Scrape Metrics from Jenkins EC2

- **Why**: Prometheus scrapes system metrics from the Jenkins EC2 (through Node Exporter) for monitoring purposes. The `prometheus.yml` file needs to be updated to include the private IP of the Jenkins EC2 as a target to ensure Prometheus pulls data from it. By default, Node Exporter exposes metrics on Port 9100, hence why we had to add an Inbound Rule to our Jenkins EC2 security group to allow traffic on Port 9100. Without this rule in place, Prometheus would be unable to collect the metrics exposed by Node Exporter. 

- **How**:
    1. **Edit the `prometheus.yml` file**:
       ```bash
       sudo nano /usr/local/prometheus/prometheus.yml
       ```
       Add the following section under `scrape_configs` to target the Jenkins EC2:
       ```yaml
       scrape_configs:
         - job_name: 'jenkins'
           static_configs:
             - targets: ['<Pivate_IP_Of_Jenkins_EC2>:9100']
        ```
    2. **Restart Prometheus** to apply the changes:
       ```bash
       sudo systemctl restart prometheus
       ```

---
### Add Prometheus as a Data Source in Grafana and Create Dashboards

- **Why**: Once Prometheus is scraping metrics, Grafana provides a user-friendly way to visualize the data. Creating a dashboard with graphs of system metrics (like CPU usage, memory usage, etc.) enables easy monitoring and helps track the health of the Jenkins EC2 in real time. By visualizing metrics in Grafana, we can track system health and detect anomalies or resource exhaustion on the Jenkins EC2 in real time. This ensures that Jenkins operates smoothly and that any issues are quickly identified before they impact the application's performance or availability.

- **How**:
    1. **Add Prometheus as a data source in Grafana**:
       - Open Grafana in the browser: `http://<MONITORING_EC2_PUBLIC_IP>:3000`
       - Login with default credentials (`admin/admin`).
       - Navigate to **Configuration > Data Sources**, click **Add data source**, and select **Prometheus**.
       - In the **URL** field, enter: `http://localhost:9090` (since Prometheus is running locally on the Monitoring EC2).
       - Click **Save & Test**.

    2. **Create a dashboard with relevant graphs**:
       - Go to **Dashboards > New Dashboard**.
       - Select **Add new panel**, and choose **Prometheus** as the data source.
       - For each graph, write Prometheus queries like:
         - CPU Usage: `node_cpu_seconds_total`
         - Memory Usage: `node_memory_MemAvailable_bytes`
       - Save the dashboard with an appropriate name (e.g., **Jenkins Monitoring**).

---
## SYSTEM DESIGN DIAGRAM

[The System Architecture Diagram for the Microblog Application can be found here](/diagram.jpg)

---
## ISSUES/TROUBLESHOOTING

### Issue: Jenkins Build Fails Due to Virtual Environment
- **Problem**: Jenkins build failed during the `Build` stage with a `bad interpreter: permission denied` error. This presented a subtantial roadblock for me early in the workload's lifepsan, as I could not for the life of me figure out what the `bad interpreter: permission denied` was referring to. I tried removing and reinstalling Python and all it's dependencies, changing Jenkins ownership permissions of both the workload's source code directory and the project workspace created by Jenkins during the Build stage, and even terminated and rebuilt the EC2 twice to wipe away any additional issues I may have introduced with my myriad attempts at resolving the issue, to no avail. 
  
- **Solution**: After terminating and recreating the Jenkins EC2 for the third time, I took a step back, took my nose off the grindstone, and took stock. What could possibly be introducing the same problem in three different EC2's? The problem was with my GitHub repository. In my testing and launching of the Microblog Flask Application from the Jenkins EC2 Terminal prior to attempting the Jenkins build, I had created a `venv` in the Workload source code and inadvertently pushed that `venv` to GitHub, committing it to the repo. This conflicted with Jenkins attempts to create it's own `venv` in the workspace directory created during a Jenkins build, this leading to the `bad interpreter: permission denied` issue. The solution was to delete the venv directory from the GtitHub repo and exclude any future `venv` directory from version control (using `.gitignore`), thus ensuring Jenkins was able to create the virtual environment and install the necessary dependencies without conflict.

---
### Issue: PYTHONPATH Not Set for Jenkins Test Stage
- **Problem**: Another roadblock for me was ensuring Jenkins could successfully use pytest to test my `test_app.py` script and thus pass the `Test` stage. Jenkins was unable to run any of the tests in my test script because it kept running into a `Module not Found` error when beginning the test. My test script is in a separate directory from the application module (`microblog.py`), so python had to be told explicitly where to look for the application module. To accomplish this, the PYTHONPATH environment variable had to be set.
  
- **Solution**: I declared the `PYTHONPATH` variable in the Jenkinsfile's `Test` stage, directly before Jenkins activates the venv in it's workspace. Since the venv and the application module are in the same directory, I set the Python path as `PYTHONPATH="."`. This allowed the testing script to find and import the application module and run the tests, allowing the Jenkins build to move to the next stage of the jenkinsfile - where my biggest roadblock yet lied in wait!

---
### Issue: The Neverending OWASP Scan
- **Problem**: This workload was the first time I enocuntered the `OWASP Scan` in a Jenkisfile. I had to install it from the list of available plugins in the Jenkins UI, and configure it to check all of the dependencies the application needed to run. When running the Jenkins build, the `OWASP Scan` actually outputs a warning that the scan will take a long time, so I knew I had to exercise some patience with scan to allow it time to download the National Vulnerability Database in order to determine if any of our libraries or packages were out of date and thus posing a security risk. What I was not prepared for was the freezing, unresponsiveness and crashing that this would cause for my measly t3.micro EC2 instance. At the outset of the scan, Jenkins would show that it would take around 4 hours to complete, so I waited and hoped that the freezing I saw in my Jenkins UI and EC2 instance meant that the scan was still happening in the background, where I could not see it's progress. This was not the case.

- **Solution**: The OWASP scan required more memory than my t3.micro EC2 instance could allot it in order to complete the `OWASP Scan` stage, and the feezing would invaribly lead to a hard crash of the EC2 instance, and thus the Jenkins build, after hours of supposed progress. I would reboot the instance, log back into Jenkins, and attempt to restart the build at the OWASP Scan phase, to no avail - it would always crash before completion. The only solution was to upgrade the EC2 instance to a t3.medium, which more than tripled the available memory for the `OWASP Scan` to use, allowing the Scan stage to complete (and pass with flying colors!) in an hour and a half. Upgrading your EC2 instance is not a decision you should make likely - it will incur additional costs for your project, so you must make sure dependency and assured security for higher cost is a tradeoff you are willing to make before upgrading. 

---
### Issue: Handling Gunicorn in the Deploy Stage
- **Problem**: The Jenkins build and it's associated Jenkinsfile posed a number of challenges for me during this workload, much more than the previous two, and this was (thankfully) the final issue I grappled with. I erroneously used the same command I had used during my manual launch of the Microblog Flask application in the Jenkins EC2 terminal as the command to deploy the application in the Jenkinsfile: `gunicorn -b :5000 -w 4 microblog:app`. There are two issues with deploying the application with this command : 1) The Jenkins build will never end, because the gunicorn process is meant to listen in for any incoming requests, effectively running indefinitely. iF the process is not set to run in the background, then it will lock up the terminal, and thus lock up the Jenkins build. Appending an '&' at the end of the command will allow it to run in the background, but then the second issue occurs. 2) The Gunicorn process ends when the Jenkins build completes because Jenkins kills all processes used during the pipeline, and the Flask app is no longer served. Leading to a 502 NGINX error when you attempt to visit the website. So I had to find a new solution. Can you guess what it is? 
  
- **Solution**: The solution was already spoiled above, outlined in the steps portion: I had to create a Gunicorn daemon (a service file) to control the Gunicron process, turning it into a service that can be started alongsided the EC2. This allows us to use systemctl commands to control the service, such as starting it, stopping it, and checking it's status. For our Jenkinsfile, the most important command it gives us is `sudo systemctl restart gunicorn`. This restars the Gunicorn serivice upon ending the Jenkins build, allowing it to run indefinitely once Jenkins has completed the build stage. In conjunction with the solution, Jenkins must also be added as a Sudoer so it can execute sudo commands, which is necessary because service files exists in the root directory of Linux systems (in this case, the path is `/etc/systemd/system/`). By adding the `sudo systemctl restart gunicorn` command to our `Deployment` stage, we are finally able to deploy the Microblog Flask Application.

---
## Configuring Node Exporter as a Scrape Target 
- **Problem:** My final pain point with the deployment of the Microblog Flask Application and the set up of the Monitoring apparatus was with the configuration of Node Exporter as a target for Prometheus to scrape. I initially used my Jenkins EC2's Public IP as the target for the scrapping. The issue with this is that the PUblic IP changes every single time the EC2 is stopped and started again. I cannot leave the EC2 on indefinitely, so a solution had to be found.

- **Solution:** The solution was to use the Jenkins EC2's **Private IP** instead of the Public IP. The Private IP remains the same regardless of how many the times the EC2 is stopped and restarted. This dovetails perfectly with the lessons I have been learning about VPC's - since both of the EC2's exists within the smae VPC, they are able to communicate to eachother via their Private IP's, without needing internet access via their Public IP's to do so. We still need to configure an Inbound Rule for the Jenkins EC2 to allow Incoming Traffic over Port 9100 because the security group acts as a firewall, and even though internet factor does not factor in to the scrappin of metrics, Prometheus still needs to be allows to scrape the exposed metrics on Port 9100 by the firewall.

---  
### OPTIMIZATION

#### **Advantages of Provisioning Our Own Infrastructure**
**Why this is a "Good System"**
1. **Full Control Over Infrastructure**: Provisioning our own resources allows for granular control over every aspect of the infrastructure. From configuring the EC2 instances, installing services like Jenkins, Prometheus, and Grafana, to managing security groups and networking. This level of control offers flexibility, especially in troubleshooting and custom configuration that might not be possible with managed services like Elastic Beanstalk.
   
2. **Cost Efficiency**: By manually provisioning resources, you can optimize your spending. You can choose smaller instance types, manage start/stop schedules, and ensure you're only paying for what you actually need. With managed services, you're paying for convenience, but sometimes at a higher cost.

3. **Customization**: This system allows the integration of tools like Prometheus and Grafana for monitoring, a custom CI/CD pipeline using Jenkins, and the flexibility to adjust configurations as per the workload's unique requirements. Such customization might not be as easy or even possible with managed services.

4. **Enhanced Learning and Skill Development**: Building infrastructure from scratch strengthens fundamental DevOps skills, including cloud infrastructure management, networking, system monitoring, and CI/CD pipeline creation. This project provided hands-on experience with AWS, system architecture, monitoring, and deployment strategies.

---

#### **Disadvantages of Provisioning Our Own Infrastructure**
**Why this could be a "Bad System"**
1. **Increased Management Overhead**: Provisioning our own resources requires constant oversight. We must handle patching, updates, monitoring, and scaling on our own. This can lead to inefficiencies and missed opportunities for automated optimizations that managed services provide out-of-the-box.

2. **Scalability Concerns**: While we can scale manually by adding more instances or adjusting instance types, the lack of automatic scaling might lead to resource bottlenecks or underutilization during varying traffic loads. Elastic Beanstalk or other managed services provide auto-scaling, making it easier to handle fluctuating workloads without intervention.

3. **Risk of Misconfiguration**: As we manage all components ourselves, there is a higher chance of misconfigurations that could compromise security, stability, or performance. For example, improperly configured Security Groups or network permissions could expose the system to unnecessary risk.

4. **Time-Consuming Setup**: Deploying an entire infrastructure manually, including monitoring and CI/CD pipelines, takes significantly more time compared to using Elastic Beanstalk or a similar service where much of the setup is automated.

---

#### **How to Optimize the System to Address These Issues - Managed Services Edition**
1. **Introduce Automation**:
   - **Infrastructure as Code (IaC)**: We could tilize tools like **Terraform** or **AWS CloudFormation** to automate the provisioning of resources. This would reduce manual effort and ensure consistency across deployments.
   - **CI/CD Automation**: Further optimize the Jenkins pipeline by adding automatic triggers for scaling the infrastructure based on demand (e.g., leveraging AWS Lambda functions or CloudWatch Alarms to provision resources as needed).
   
2. **Auto-Scaling and Load Balancing**:
   - Incorporate **Auto Scaling Groups (ASGs)** and **Elastic Load Balancers (ELBs)** to ensure that the infrastructure can handle varying traffic loads dynamically. This would eliminate the need for manual scaling and reduce the risk of bottlenecks or downtime during traffic spikes.
   
3. **Security Best Practices**:
   - Implement more robust security practices, such as **AWS Secrets Manager** to handle sensitive data and **AWS Key Management Service (KMS)** to encrypt data in transit and at rest.
   - Audit the **Security Groups** and **IAM roles** to ensure the principle of least privilege is being followed.

4. **Monitoring and Alerting**:
   - While Prometheus and Grafana offer great monitoring, adding **AWS CloudWatch** or similar monitoring services could provide additional real-time insights and more automated alerting capabilities.
   - Set up custom **CloudWatch alarms** that can trigger based on resource utilization metrics or any performance anomalies detected through Prometheus.

5. **Consider Managed Services for Certain Components**:
   - For example, using a managed database service like **Amazon RDS** or **AWS Elasticache** can offload the management of database scaling, backups, and patching. Similarly, **AWS Fargate** or **Elastic Beanstalk** could handle scaling and deployment of the application without needing to manage EC2 instances directly.

---

### Additional Optimizations Without Full Reliance on Managed Services

1. **Use Docker Containers**: Containerizing the Microblog application with Docker can help streamline deployments and improve consistency across environments. Containers can be orchestrated with tools like **Docker Compose** or **Kubernetes** without fully relying on managed services.

2. **Caching Strategies**: Implement caching mechanisms using tools like **Redis** or **Memcached** to reduce load on the database and speed up response times without relying on managed caching services.

3. **Regular Backups**: Implement a backup strategy using tools like **rsync** or **cron jobs** to automate backups of application data and configurations, ensuring that you can restore quickly in case of failure without relying on managed backup solutions.

By implementing these optimizations, we would maintain the benefits of provisioning our own infrastructure while addressing many of the shortcomings that come with manual management.

## Conclusion

The successful deployment of the Microblog application on self-provisioned infrastructure marks a significant step in our understanding of cloud computing and application deployment. By leveraging EC2 instances, we not only gained hands-on experience with essential tools such as Jenkins, Prometheus, and Grafana, but we also explored the intricacies of managing our own infrastructure.

Throughout this project, we faced various challenges, from configuring security groups to optimizing build processes. Each issue encountered served as a learning opportunity, enhancing our problem-solving skills and deepening our knowledge of CI/CD pipelines.

By implementing a monitoring solution with Prometheus and Grafana, we ensured the reliability of our application, providing real-time insights into system performance. This project has demonstrated the importance of having a robust infrastructure, tailored to our specific needs, while also highlighting the advantages of understanding the underlying technologies rather than solely relying on managed services.

As we move forward, the skills and knowledge gained from this workload will be invaluable in our future endeavors in cloud computing and application development. We are now better equipped to tackle complex deployments and optimize system performance, laying a strong foundation for further exploration in the world of DevOps.


## Documentation

*****

**Inbound Rules for Jenkins & Monitoring Security Groups**

![image](https://github.com/user-attachments/assets/32a7c388-3ce9-44d0-9d9f-9c67ea5c87b4)

**Grafana Visualization of Jenkins EC2 Metrics**

![image](https://github.com/user-attachments/assets/9848df17-a3fe-4838-9de9-ed9ce9b0b36e)

**Microblog Login page**

![image](https://github.com/user-attachments/assets/3479abdf-e2a3-474a-849c-8b5829b5112b)

**Successful OWASP Dependency Check**

![image](https://github.com/user-attachments/assets/59b6ae5b-6d8f-41e8-a216-a752cab49bac)

**Successful Pytest**

![image](https://github.com/user-attachments/assets/df888839-c8eb-4f07-ba06-e2ac5fd0ac8b)
