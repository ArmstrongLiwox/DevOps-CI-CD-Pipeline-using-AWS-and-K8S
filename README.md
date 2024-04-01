# **DevOps CI CD Pipeline using AWS and K8S**

In this live project, I will create a complete DevOps project using
Jenkins, SonarQube, Trivy, Docker, and Kubernetes.

This CI/CD pipeline will work in a completely automated manner; any
change on the GitHub repository will trigger the pipeline. Once pipeline
execution is complete, you will see the changes appearing on the
application, which we will deploy on Kubernetes.

## **Project Summary**

![project architecture](images/project.png)
---
The image above defines the architecture of this project. So lets explain it. 
First of all, we will use Terraform to create the instance for Jenkins,
and all the packages we will install in that instance using Terraform
itself. In our CI/CD pipeline, if a user makes any change on the GitHub
repository, it will trigger the pipeline. Our pipeline will start
executing the stages. First of all, it will do the SonarQube analysis of
our code, then it will perform the npm dependencies installation, and
then it will do the Trivy scan of our code. Next, it will build the
Docker image and push that Docker image to the Docker Hub. Then, it will
scan that Docker image using Trivy. Finally, our Jenkins script will
deploy the pods on Kubernetes using the Docker image, and we will have
the monitoring configured using Prometheus and Grafana, which will
monitor the EKS cluster. It will also monitor Jenkins. After completion
of every build, we will get the email notification on our Gmail ID with
the Trivy scan results and the complete log of the job completed.

**So let\'s start building our project.**

On my Windows 10 system I have Termius (I can also use Gitbash
terminal), Visual Studio Code and the AWS CLI installed.

You can download and install to follow along.


I will create the folder for my project, and inside it, I will create
one more folder called \"Jenkins-SonarQube-VM.\"
![1](images/image1.png)

### Create main.tf file

On the Visual Studio Code, I will open the folder which I have created,
and on this folder \"Jenkins-SonarQube-VM,\" I will create one file
\"main.tf,\".

![2](images/image2.png)

I will paste this content for the main.tf 
```
resource "aws_instance" "web" {
  ami                    = "ami-080e1f13689e07408"      
  instance_type          = "t2.large"
  key_name               = "dockerize"            
  vpc_security_group_ids = [aws_security_group.Jenkins-VM-SG.id]
  user_data              = templatefile("./install.sh", {})

  tags = {
    Name = "Jenkins-SonarQube"
  }

  root_block_device {
    volume_size = 40
  }
}

resource "aws_security_group" "Jenkins-VM-SG" {
  name        = "Jenkins-VM-SG"
  description = "Allow TLS inbound traffic"

  ingress = [
    for port in [22, 80, 443, 8080, 9000, 3000] : {
      description      = "inbound rules"
      from_port        = port
      to_port          = port
      protocol         = "tcp"
      cidr_blocks      = ["0.0.0.0/0"]
      ipv6_cidr_blocks = []
      prefix_list_ids  = []
      security_groups  = []
      self             = false
    }
  ]

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "Jenkins-VM-SG"
  }
}

```

### lets explain the main.tf file

The main.tf file will contain details for the infrastructure I intend to
provision on my AWS console like:

-   The AMI; you need to change it as per your region.

-   Instance type I\'m going to create T2 large.

-   key name,

-   Name for the Linux server.

-   The user data file \"install.sh,\" which it will run inside the
    created EC2 instance.

-   Name to the EC2 instance \"Jenkins-SonarQube.\"

-   Volume size of which we will be using 40gb,

-   The security group.

I will log in to my AWS to get some data (AMI for my instance) to update my main.tf file.

![4](images/image4.png)


We will use the default VPC for this project. So we don\'t need to create the custom VPC.

![5](images/image5.png)


Once again, it will just associate these Security Groups to the default VPC of the account, and it is also going to open the inbound ports 22, 80, 443,

8080 for the Jenkins,

9000 for the SonarQube, and

3000 for our application.

I will save this file. 

I will now create one more file \"provider.tf,\" and I will provide the content in this file.

```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
# Configure the AWS Provider
provider "aws" {
  region = "us-east-1"     #change region as per you requirement
}

```

![6](images/image6.png)


Here, actually, we are defining, we are registering the provider, which
is the AWS. This is my region us-east-1. I will save this file.

And finally, I will create the \"install.sh\" file, which it is going to run inside the created EC2 instance.

```
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins

##Install Docker and Run SonarQube as Container
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu
sudo usermod -aG docker jenkins  
newgrp docker
sudo chmod 777 /var/run/docker.sock
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

#install trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```
![7](images/image7.png)

### lets explain the install file

So, this \"install.sh\" file will first update the packages, then it
will install the JDK, then it will install the Jenkins in our EC2
instance.

It will then it will install the Docker in that instance, and then it
will run the SonarQube as the container in the EC2 instance.

This is the image name for the SonarQube

docker run -d \--name sonar -p 9000:9000 sonarqube:lts-community

Which is the standard image of the SonarQube.

And after that, it will install the Trivy in that EC2 instance. I will
save this file also.

Note that The path has been specified within our main.tf file.

### **Use AWS CLI**

On the terminal.I already have my AWS credentials. I will check the
users I have.

![8](images/image8.png)


I will give the command \"***AWS configure***\". I will paste the access
key and hit enter. I will paste the secret key also and hit enter. So,
we have successfully logged in from the terminal.

![9](images/image9.png)


Now I will check for terraform on my PC. It shows its outdated so I will download the recent and check again. 

So now I will use gitbash because my VScode integrated terminal is not working.

![10](images/image10.png)


On the VS Code, I\'m inside this folder. I will give the command

\"***terraform init***.\"

![11](images/image11.png)


Okay, so Terraform has been initialized in this folder. I will now give
the command

\"***terraform plan.\"***

![12](images/image12.png)


Okay, so it is going to create these resources in our AWS account.

Finally, I will give the command \"***terraform apply
-auto-approve***.\"

![13](images/image13.png)


Instance created. If I go to the AWS console, instances, the region us-east-1, we will see the instance which has been created, the security group and the inbound rules which have been applied as per the Terraform script.

![14](images/image14.png)


I will then SSH into my instance by clicking on connect and copying the SSH string and paste into my git bash terminal.

![15](images/image15.png)

![16](images/image16.png)


I am inside my ec2 instance now. I will give the command
\"***jenkins-version***,\" so Jenkins has been installed on my system. I will give the command \"***docker-version***,\" so Docker is also
installed.

I will also give the command \"***trivy-version***,\" so this is the version of the Trivy which has been installed on my system. If I give the command \"**docker-ps -a**,\" we will see the container for the SonarQube which is running. Created about a minute ago.

![17](images/image17.png)


We are actually running the SonarQube as the container on this EC2
instance. 

>You might be doing this project in multiple settings, so while
you are shutting down your EC2 instance, you need to first stop the
Docker container first. Give this command \"***docker ps -a***,\" you
will get the ID for the container, and then give the command \"***docker
stop***\" and the container ID. After that, you can shut down this EC2
instance, and once you are back, you need to give give the command
\"***docker start***\" and the container ID to restart the SonarQube.

## **Configure Jenkins**

We have installed the EC2 instance for Jenkins and SonarCube. Now we
need to configure Jenkins on that EC2 instance. I will browse the public
IP of the instance with port 8080, go to this path on the terminal, and
copy the suggested password for installation.

![18](images/image18.png)


Install suggested plugins and I will create the user \"Cloud admin\" and
set the password.

![19](images/image19.png)

![20](images/image20.png)

![21](images/image21.png)

![22](images/image22.png)

![23](images/image23.png)

Okay, so we are inside the Jenkins dashboard now. We need to install a few plugins required for our project. 

I will go to \"Manage Jenkins\" \> \"Manage Plugins\" \> \"Available\" and install the following plugins:

\- Eclipse Timing Installer

\- SonarQube Scanner

\- Sonar Quality Gates

\- NodeJS

\- Docker (Docker Commons, Docker Pipeline, Docker API, Docker Build
Step)

![24](images/image24.png)

![25](images/image25.png)


Now we will install some tools required for this project. I will go to
\"Manage Jenkins\" \> \"Global Tool Configuration\" and add the
following tools:

\- NodeJS 16

\- JDK 17

\- Docker

![26](images/image26.png)

![27](images/image27.png)

![28](images/image28.png)

![29](images/image29.png)


Next, under \"Manage Jenkins,\" I will add the SonarCube server with the
appropriate settings.

![30](images/image30.png)


## **Configure SonarCube**

Now it\'s time to configure SonarCube. I will copy the public IP of the
instance and browse it with port 9000.

![31](images/image31.png)


Provide the default credentials, set a new password, and update tokens
for Jenkins.

![32](images/image32.png)

![33](images/image33.png)


Then I will create a token for SonarCube and add it to Jenkins
credentials.

![34](images/image34.png)

![35](images/image35.png)

![36](images/image36.png)

![37](images/image37.png)

![38](images/image38.png)


Finally, we need to integrate SonarCube with Jenkins by adding the
SonarCube server under \"Manage Jenkins\" \> \"Configure System.\"

![39](images/image39.png)


Go to quality gates on SonarCube dashboard and create

![40](images/image40.png)

![41](images/image41.png)

![42](images/image42.png)


Now we have to create the webhook between SonarCube abd Jenkins.

![43](images/image43.png)

![44](images/image44.png)

![45](images/image45.png)

![46](images/image46.png)


We have configured Jenkins and SonarCube and integrated them. Now it\'s time to create the pipeline. First, I will create a token for our
project in SonarCube. Then, in the Jenkins dashboard, I will create a new item with the required pipeline script.

![47](images/image47.png)

![48](images/image48.png)

![49](images/image49.png)

![50](images/image50.png)

![51](images/image51.png)

![52](images/image52.png)


## **Create pipeline in Jenkins**

![53](images/image53.png)

![54](images/image54.png)


> Click on Discard old builds

> Max number of builds to keep 2

![55](images/image55.png)

![56](images/image56.png)

![57](images/image57.png)

![58](images/image58.png)

![59](images/image59.png)

![60](images/image60.png)


## **Create pipeline**

Next, we need to configure our pipeline to build a Docker image and push
it to Docker Hub. For this, we need to create a personal access token on
Docker Hub and add it to Jenkins credentials. Then, we add stages in our
Jenkins pipeline script to build, tag, push, and scan the Docker image.

![61](images/image61.png)

![62](images/image62.png)

![63](images/image63.png)

![64](images/image64.png)

![65](images/image65.png)

![66](images/image66.png)


![](vertopal_eb288dc93fd54abc85691594073eb467/media/image67.png){width="7.5in"
height="3.8569444444444443in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image68.png){width="7.5in"
height="3.9208333333333334in"}

**Set up Prometheus and Grafana**

In this section of the project, we will set up monitoring with
Prometheus and Grafana by creating a separate instance. Using Terraform,
we create the necessary resources and run commands to install
Prometheus, node exporter, and Grafana. After setting up, we configure
Prometheus to include the node exporter job and check Grafana to
visualize the metrics.

Create a main.tf file

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image69.png){width="7.5in"
height="4.084027777777778in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image70.png){width="7.5in"
height="4.013194444444444in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image71.png){width="7.5in"
height="4.143055555555556in"}

Terraform init

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image72.png){width="7.5in"
height="4.1305555555555555in"}

Terraform plan

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image73.png){width="7.5in"
height="4.083333333333333in"}

terraform apply -auto-approve

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image74.png){width="7.5in"
height="4.201388888888889in"}

Check AWS to see new instance

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image75.png){width="7.5in"
height="3.7666666666666666in"}

Access new instance

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image76.png){width="7.5in"
height="3.9097222222222223in"}

sudo systemctl status Prometheus

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image77.png){width="7.5in"
height="3.783333333333333in"}

sudo systemctl status node_exporter

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image78.png){width="7.5in"
height="3.761111111111111in"}

sudo systemctl status grafana-server

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image79.png){width="7.5in"
height="3.8256944444444443in"}

Go to public IP on port 9090

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image80.png){width="7.5in"
height="3.854861111111111in"}

On the terminal go to

cd /etc/prometheus

sudo nano prometheus.yml

add node-exporter job

static_configs:

\- targets: \[\"localhost:9090\"\]

\- job_name:\'node_exporter\'

static_configs:

\- targets: \[\"54.165.191.30:9100\"\]

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image81.png){width="7.5in"
height="3.9347222222222222in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image82.png){width="7.5in"
height="3.861275153105862in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image83.png){width="7.387491251093613in"
height="3.863888888888889in"}

Check indentation promtool check config /etc/prometheus/prometheus.yml

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image84.png){width="7.5in"
height="3.8125in"}

Reload the Prometheus configuration

curl -X POST http://localhost:9090/-/reload

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image85.png){width="7.5in"
height="3.8506944444444446in"}

Check for node exporter on dashboard

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image86.png){width="7.5in"
height="3.8270833333333334in"}

Access grafana with the public IP on port 3000

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image87.png){width="7.5in"
height="3.8895833333333334in"}

Change default password

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image88.png){width="7.5in"
height="3.8694444444444445in"}

Login

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image89.png){width="7.5in"
height="3.8875in"}

We have to set the data source

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image90.png){width="7.5in"
height="3.8682425634295714in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image91.png){width="7.5in"
height="3.9347222222222222in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image92.png){width="7.5in"
height="3.9097222222222223in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image93.png){width="7.5in"
height="3.911111111111111in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image94.png){width="7.5in"
height="4.024305555555555in"}

Load and select the source which we have created: Prometheus.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image95.png){width="7.5in"
height="3.917361111111111in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image96.png){width="7.5in"
height="3.908333333333333in"}

Import so we can see the dashboard for our monitoring server using the
Node Explorer. Multiple dashboards are available.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image97.png){width="7.5in"
height="3.9444444444444446in"}

Multiple dashboards are available

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image98.png){width="7.5in"
height="3.9256944444444444in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image99.png){width="7.5in"
height="4.009027777777778in"}

Now we need to integrate Jenkins with Prometheus so that we can import
the dashboard for Jenkins also on Grafana. So I will go to Jenkins -\>
Manage Jenkins -\> Available plugins. I will search for \"Prometheus\"
and select the plugin \"Prometheus Matrix\" and click on install. It is
asking to restart, so I will restart Jenkins.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image100.png){width="7.5in"
height="3.8270833333333334in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image101.png){width="7.5in"
height="3.8583333333333334in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image102.png){width="7.5in"
height="3.8979166666666667in"}

Login again, go to Manage Jenkins -\> System, and under the system, the
Prometheus path is Prometheus. I will take these two options also and
the job name is Jenkins job. I will click on apply and save.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image103.png){width="7.5in"
height="3.942361111111111in"}

I will go to the terminal of the server and again go to CD -\> Etc -\>
Prometheus. I will open the prometheus.yml file and add the job for the
Jenkins job. I will save this file.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image104.png){width="7.5in"
height="3.8958333333333335in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image105.png){width="7.5in"
height="3.9090277777777778in"}

I will run the command to check the indentation of the YAML file.

promtool check config /etc/prometheus/prometheus.yml

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image106.png){width="7.5in"
height="3.86788823272091in"}

Now it is successful. I will reload this service. If I go to Prometheus
dashboard and do the refresh, Jenkins job is added. Data fetching may
take some time. See the target is up, and the log is coming.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image107.png){width="7.5in"
height="3.917361111111111in"}

Now we need to add the dashboard for Jenkins on Grafana. I will go to
Grafana dashboard and search for \"import dashboard\" and type the
dashboard ID 9964.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image108.png){width="7.5in"
height="4.083333333333333in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image109.png){width="7.5in"
height="3.9652777777777777in"}

I have given the ID for the dashboard. I will load it. The name of the
dashboard is taken. I will select the source Prometheus and import. This
is the dashboard for the Jenkins executor.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image110.png){width="7.5in"
height="4.027083333333334in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image111.png){width="7.22336832895888in"
height="3.775in"}

Free to go to Jenkins. I will go to job and I will click on build.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image112.png){width="7.5in"
height="3.8944444444444444in"}

Now the job is completed. If I go to Grafana dashboard and do the
refresh, the successful job is showing as one. After integrating Jenkins
with Grafana, we have the successful job quantity as one.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image113.png){width="7.5in"
height="3.9229166666666666in"}

**Send email notification through jenkins**

In this section of the project, we need to create the email alert for
our job in Jenkins. So I have logged into my Google account and I\'m
inside my account.google.com.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image114.png){width="7.5in"
height="3.9in"}

Your account must have the two-factor authentication enabled. I will go
to Security -\> App password. I will give the name and click on create.
I will copy this to my system and close.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image115.png){width="7.5in"
height="3.907638888888889in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image116.png){width="7.344555993000875in"
height="3.7472222222222222in"}

Now we need to enter these credentials into Jenkins. So I will go to
Jenkins -\> Manage Jenkins -\> Manage credentials. Add credentials -\>
Username with password. Username is my Gmail address and I will provide
the created password. Here ID I will give Gmail, description I will give
Gmail and I will click on create.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image117.png){width="7.5in"
height="3.897222222222222in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image118.png){width="7.5in"
height="3.8652777777777776in"}

Now I will go to Manage Jenkins -\> System and I will go to Email
settings. SMTP server I will provide smtp.gmail.com. Default user I will
provide my email address. I will go to Advanced -\> Use SMTP
Authentication. Username I will provide my username and password which
we have created.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image119.png){width="7.340788495188101in"
height="3.8743055555555554in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image120.png){width="7.4998501749781274in"
height="3.873611111111111in"}

I will again click on test configuration. Email was sent successfully.
If I go to my Gmail, I have received the test email from the Jenkins.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image121.png){width="7.5in"
height="3.911111111111111in"}

I will now go to Extended Email Notification -\> Server. I will give
SMTP.gmail.com, Port 465, Advanced. Credentials I will select the
credentials which we have created (Gmail). Use SSL, default content type
I will select HTML. Under the triggers, I will select \"always\" and
\"success\".

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image122.png){width="7.5in"
height="3.8743055555555554in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image123.png){width="7.5in"
height="3.966666666666667in"}

Apply and then save.

Now we need to modify the pipeline script and we need to give the post.
So I will go to my pipeline -\> Configure. I will go to Script and after
the stages, I will provide the post.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image124.png){width="7.5in"
height="3.9270833333333335in"}

I will click on apply and save. I will click on build.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image125.png){width="7.5in"
height="3.970833333333333in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image126.png){width="7.5in"
height="3.8916666666666666in"}

Now the job completed successfully and if I go to my Gmail account, I
have received the email from the pipeline which has the complete build
log along with the Trivy file scan result and Trivy image scan
result.![](vertopal_eb288dc93fd54abc85691594073eb467/media/image127.png){width="7.5in"
height="3.9506944444444443in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image128.png){width="7.5in"
height="3.941666666666667in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image129.png){width="7.5in"
height="3.9243055555555557in"}

**Configure AWS EKS cluster**

In this section of the project, we are going to configure the AWS EKS. I
have taken the remote of the Jenkins SonarQube server. First of all, I
need to install the kubectl in this server. So I will give the command
to install the URL. Then I will download the package for the kubectl,
then I will install it. If I give the command kubectl version, this is
the version of the kubectl which has been installed.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image130.png){width="7.5in"
height="3.8118055555555554in"}

Now I will go ahead and install the AWS CLI in this server. First of
all, I will download the packages for the AWS CLI. I will install the
unzip in this server. I will unzip the downloaded package and then I
will install the executable file. If I give the command AWS \--version,
this is the version of the AWS CLI installed in my system.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image131.png){width="7.5in"
height="3.77245406824147in"}

Now we need to install the eksctl. So first of all, I will download the
packages for the eksctl using this command ( ). These packages have been
downloaded in the TMP folder. So I will go to the TMP directory and I
will move the executable file to the bin folder because all the
executable files are under the bin. If I give the command eksctl
version, eksctl has been installed in my system.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image132.png){width="7.5in"
height="3.8090277777777777in"}

Now we need to create the IAM role and attach it to the EC2 instance. So
on my AWS console, I will go to IAM -\> Roles -\> Create Role. AWS
service in the dropdown, I will select EC2. Next, I will select the
AdministratorAccess. Next, I will give the name \"eksctl_role\". Create.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image133.png){width="7.5in"
height="3.8520833333333333in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image134.png){width="7.5in"
height="3.877083333333333in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image135.png){width="7.5in"
height="3.8493055555555555in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image136.png){width="7.5in"
height="3.8541666666666665in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image137.png){width="7.5in"
height="3.8305555555555557in"}

So the role has been created. I will go to my EC2 instance, I will go to
Action -\> Security -\> Modify IAM role and in the dropdown, I will
select the created role. Update IAM role.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image138.png){width="7.5in"
height="3.7243055555555555in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image139.png){width="7.5in"
height="3.7666666666666666in"}

Okay, so role has been assigned to the EC2 instance.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image140.png){width="7.5in"
height="3.772222222222222in"}

Now I will go ahead and create the EKS cluster using the eksctl. So I
will give the command in which this would be the name of my EKS cluster.
I will hit enter, then I will select my region AP South one, then I will
give the node type T2 small, then I will give the node count 3.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image141.png){width="7.5in"
height="3.6659722222222224in"}

This Kubernetes cluster creation will take some time. Kubernetes cluster
creation completed.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image142.png){width="7.5in"
height="3.6597222222222223in"}

If I give the command kubectl get nodes, these are three nodes created
in my cluster. If I give the command kubectl get SVC, this is the
default service of my Kubernetes cluster.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image143.png){width="7.5in"
height="3.8645833333333335in"}

**Integrate Prometheus with EKS and Import Grafana Monitoring Dashboard
for Kubernetes**

Now we need to configure the monitoring for our Kubernetes cluster. So
we will install the Prometheus on the EKS cluster and then we will add
the Prometheus as a source to the Grafana dashboard. So to install the
Prometheus, first of all, we need to install the Helm on the server. So
I will install the Helm. If I give the command helm \--version, so the
Helm has been installed in my system.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image144.png){width="7.5in"
height="3.7333333333333334in"}

Now we need to add the Helm stable charts for our local client. So for
that, I will give the command. Then I will add the Prometheus Helm repo.
Then I will go ahead and create the separate namespace for the
Prometheus and then finally, I will go ahead and install the Prometheus
using Helm. Okay, so it has been installed. I will now give the command
to check if Prometheus has been installed or not. If I give the kubectl
command, these are the pods created for the Prometheus. Then I will
check the service which has been created for the Prometheus. So this is
the service created for the Prometheus, but these are not the load
balancer

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image145.png){width="7.5in"
height="3.888888888888889in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image146.png){width="7.5in"
height="3.736111111111111in"}

So I will give you the cube CDL command \`ql get PS\` namespace
Prometheus so these are the ports created in the for the Prometheus.
Then I will check the service which has been created for the Prometheus
so this is the service created for the Prometheus.

But these are not the load balancer service so these are not exposed to
the external world so to expose the service to the external world I will
open the file and in this file I will go to end and here type is given
as cluster IP

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image147.png){width="7.204304461942257in"
height="3.7243055555555555in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image148.png){width="7.416707130358705in"
height="3.736111111111111in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image149.png){width="7.5in"
height="3.7243055555555555in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image150.png){width="7.5in"
height="3.761111111111111in"}

So I will make it I will type I button to insert and I will make it load
balancer and then this port I will make 9090 I will go ahead and save
this file I will press escape button then colum WQ and hit enter if
again give the command to check the service so now you can see we have
the load Balan service instead of cluster IP and this is the external uh
DNS for the service I will copy this URL and on the browser I will
browse it with the port 9090.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image151.png){width="7.5in"
height="3.8027777777777776in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image152.png){width="7.5in"
height="3.970833333333333in"}

So you can see Prometheus is ready and it has been installed on the
kubernetes cluster I can check the targets here for our kubernetes
cluster.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image153.png){width="7.5in"
height="3.9715277777777778in"}

Okay now we need to add this Prometheus as the data source in the uh
grafana so I will go to our grafana server and uh on the left side I
will go to connections data sources and we have one data source which is
our Prometheus uh server which is installed in the ec2 instance I will
add one more Prometheus as the data source which has been installed on
the kubernetes cluster so I will type here I will click here add new
data source Prometheus name I will give Prometheus e and URL I will give
HTTP column sl/ and this URL of this service colum 9090 and here I will
click on Save and test.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image154.png){width="7.5in"
height="3.9520833333333334in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image155.png){width="7.5in"
height="3.7895833333333333in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image156.png){width="7.5in"
height="3.879166666666667in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image157.png){width="7.5in"
height="3.932638888888889in"}

So connection successful okay so data source has been added now we need
to import the dashboard for the kubernetes cluster so I Will Show You by
adding the two dashboards you can uh search for the dashboard ID and you
can uh import the multiple dashboards for the kubernetes so here I will
go to search and I will go to import dashboard and here I will give the
ID for one of the dashboard 15760.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image158.png){width="7.5in"
height="3.8618055555555557in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image159.png){width="7.5in"
height="3.957638888888889in"}

I will click on load so this is the name of the dashboard kubernetes
ports and the data source I will select Prometheus EAS and I will click
on import.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image160.png){width="7.5in"
height="3.845138888888889in"}

So this is the dashboard for our e data fetching may take some time and
here on the data source in the drop down we need to se select our data
source fromus e so CP usage by container CP usage by memory usage by
container all the graphs are available here if I select the name space
Prometheus so we can monitor the ports created under the Prometheus name
space I will go to home of the dashboard so these are the dashboards
which has been imported to our uh grafana I Will Show You by importing
one more dashboard which is also for the kubernetes.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image161.png){width="7.5in"
height="4.022916666666666in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image162.png){width="7.5in"
height="3.9583333333333335in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image163.png){width="7.5in"
height="3.8229166666666665in"}

So in the search bar I will go to import dashboard and here I will give
the ID 17119 and I will click on load

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image164.png){width="7.5in"
height="3.9131944444444446in"}

so name of the uh dashboard is kubernetes e cluster I will select the
data source Prometheus e and import

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image165.png){width="7.5in"
height="3.9784722222222224in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image166.png){width="7.5in"
height="3.8625in"}

So here we can monitor complete cluster so we can monitor the name space
Prometheus here you can search for the dashboard IDs and you can import
the more dashboards regarding the kubernetes cluster to the grafana so
on our grafana we have total four dashboards as of now one one is for uh
genkins and one is for node exporter which is the local host and two are
for the EKS

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image167.png){width="7.5in"
height="3.8673611111111112in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image168.png){width="7.5in"
height="3.83125in"}

**Configure Jenkins Pipeline to Deploy Application on AWS EKS**

now finally we need to configure our genin pipeline to deploy the
resources on the kubernetes so for that we need to First deploy some
plugins to the genkins I will go to man genkins plugins available
plugins and I will search here kubernetes I will select this plugin this
one also this one also this one also total four plugins I have selected
I will click on install.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image169.png){width="7.5in"
height="3.86875in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image170.png){width="7.5in"
height="3.9458333333333333in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image171.png){width="7.5in"
height="3.8722222222222222in"}

I will go to home I will go to terminal of the server and I\'m inside
the slome slubu inside you can see one uh directory do Cube I will go to
this directory and this is the config file for the kubernetes I will
right click and download it I will save it on my project folder.

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image172.png){width="7.5in"
height="3.592361111111111in"}

I will go to that folder and I will open the config file in the notepad
and I will do the save a copy s and I will give the name secret.txt

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image173.png){width="7.5in"
height="4.0875in"}

![](vertopal_eb288dc93fd54abc85691594073eb467/media/image174.png){width="7.5in"
height="4.090972222222222in"}

now we need to add these secrets to the genkins so on the genkins I will
go to manag genkins credentials add credentials kind I will select
secret file ID I will give kubernetes and I will choose the file which
we have just downloaded and the ID is kubernetes this ID we will recall
in the pipeline script create I will now go to my pipeline configure I
will go to script and here we need to add one more stage to deploy the
resources on the kubernetes so here I will add the one more stage and
the name is deploy to kubernetes for for now it\'s okay so I will delete
this command actually we need to generate this command and directory
kubernetes this is the kubernetes directory inside our GitHub GitHub
repository kubernetes because this directory is having the Manifest
files so I will generate the command command here so to generate the
command I will go to pipeline syntax which will open the new tab and
here in the drop down select the plugin which we had installed Cube CTL
credential select the kubernetes credential and remaining things don\'t
change anything generate pipeline script copy this command and paste it
on the pipeline script for for I will click on apply and save before we
verify our cicd pipeline I will make one change on my Pipeline on the
pipeline script I will make it 3 image scan apply save while pushing the
changes to the remote repository from your uh git Bash you may be needed
the GitHub personal personal access token to be provided so on the
GitHub you can go to setting developer setting personal access token
token classic and you can create your token here I have already created
the token for myself self I will clone the repository on my G bash I
will go to Repository okay so I\'m inside my repository before we verify
our de secop pipeline we need to enable the web hook for our pipeline so
I will go to configure and here I will give GitHub project I will paste
the URL for my repository and under the build trigger I will select this
option GitHub hook trigger and I will go to my repository on the GitHub
I will go to setting web hooks add web hook I will give the URL here
HTTP colon sl/ and the public IP of the Jenkins instance colon 8080 SL
GitHub hyphen web hook slash ADD web hook I will refresh here it is
showing as green means connection is successful on my project I will
apply and save okay so everything has been set up now it\'s time to
verify our Dev SEC pipeline if I go to dockerhub you can see there is no
image for YouTube clone clone app I have removed the previous image so
on my git bash if I do the ls I will modify the readme file I will make
it test 50 crl o and enter to save crol X to exit git add git commit
hyph name get push origin main so here I need to provide my token so I
have already created the GitHub token I will go to token option and I
will paste my token here so changes has been pushed to remote repository
if I go to my Jenkins job so it has triggered automatically upon the
change on the grub Repository for job completed successfully if I go to
dockerhub and do the refresh here so image has been pushed just now and
if I go to my email I have received the email

also with the log from the pipeline if I go to my on dashboard and if I
go to the ports I will select my source and the name space is default so
data is fetching for the newly created Port if I go to server and give
the command Cube CTL get service so this is the URL of my pod I will
browse it on the new tab so this is my application running this is the
YouTube clone app you can play the videos also on this app new
legislation we will verify it once again on the application uh the color
of these symbols is red we will change it to some other color and will
verify it changes or not so on the G bash I will go to to SRC directory
I will go to components and I will open the file sidebar in the Nano
editor and this red color we will make it some other I will make it blue
control o and enter to save control X to exit K add get commit and I
will I will send the changes to the remote repository get push origin
men if I go to my pipeline the job has been triggered upon the change on
the GitHub repository job completed successfully if I go to terminal and
give here the command Cube CTL get p PS so these are the new newly
created ports we will check these ports on the grafana on the grafana
dashboard uh name is spes default in the drop down I will select the Pod
this one and if I do the refresh so data is fetching for the newly
created port and if I go to my application and do the refresh here so
you can see the change has color has been changed for these icons from
red to blue by this way you can create the complete automated Dev secop
cicd pipeline which will trigger automatically upon the change on the
GitHub repository and once job is completed you will see the changes
appearing on the application follow me on the LinkedIn I post multiple
useful useful things on the LinkedIn also you can fog my repository and
create your own branch and practice the complete video during practice
if you face any kind of difficulty you can ask me in the comment section
I will definitely reply to your query hope you find this video helpful a
lot of effort went behind this video so do subscribe to my channel to
encourage my work like the video share this video with the friends and
spread the knowledge I look forward you to join me in the next video
thank you
