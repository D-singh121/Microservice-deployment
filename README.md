# Microservice Deployment
## Overview:
This repository contains instructions for deploying a set of microservices using AWS EKS and Jenkins. The deployment involves creating an EC2 instance, setting up an EKS cluster, configuring Jenkins, and deploying the microservices.

Microservices
There are 10 microservices located in different Git branches. We use a multibranch pipeline and webhook for deployment.

Inbound Port Configuration
Ensure the following ports are open:

25, 3000-10000, 80, 443, 22, 6443, 465, 27017, 30000-32767 for IPv4 everywhere.
EC2 Instance Setup
Instance Type: t2.large (2 CPUs, 8GB RAM)
Key Pair: Use microservice
Storage: 25GB
Services: EKS cluster master, Jenkins
IAM User Creation
Create an IAM user named "EKS user" with the following access keys and policies:

plaintext
Copy code
User: dee-microservice
Access Key: AKIA3FLDZWXYG6LNQW22
Secret Access Key: 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
Policies to Attach
AmazonEC2FullAccess
AmazonEKS_CNI_Policy
AmazonEKSClusterPolicy
AmazonEKSWorkerNodePolicy
AWSCloudFormationFullAccess
IAMFullAccess
Custom Policy:
json
Copy code
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
CLI Tools Installation
Install the following CLI tools on the EC2 instance:

AWS CLI
bash
Copy code
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
kubectl
bash
Copy code
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
eksctl
bash
Copy code
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
EKS Cluster Setup
Configure AWS CLI with your credentials and region (ap-south-1):

bash
Copy code
aws configure
Create EKS Cluster
bash
Copy code
eksctl create cluster --name=EKS-1 \
                      --region=ap-south-1 \
                      --zones=ap-south-1a,ap-south-1b \
                      --without-nodegroup
To delete the cluster:

bash
Copy code
eksctl delete cluster --name EKS-1 --region ap-south-1
Associate IAM OIDC Provider
bash
Copy code
eksctl utils associate-iam-oidc-provider \
    --region ap-south-1 \
    --cluster EKS-1 \
    --approve
Create EKS Node Group
bash
Copy code
eksctl create nodegroup --cluster=EKS-1 \
                       --region=ap-south-1 \
                       --name=node2 \
                       --node-type=t3.medium \
                       --nodes=3 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=DevOps \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access
To delete node groups:

bash
Copy code
eksctl get nodegroup --cluster=EKS-1	
eksctl delete nodegroup --cluster=EKS-1 --name=node2
Jenkins Setup
Install Docker
bash
Copy code
sudo apt install docker.io
sudo chmod 666 /var/run/docker.sock
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins
Install Jenkins
Install Java JDK 17:

bash
Copy code
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
Install Jenkins:

bash
Copy code
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
Start Jenkins:

bash
Copy code
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
Access Jenkins:
Open http://<your-instance-public-ip>:8080 and use the initial password from:

bash
Copy code
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Install Plugins
Docker, Docker Pipeline
Kubernetes, Kubernetes CLI
Multibranch Scan Webhook Trigger
Configure Plugins
Configure Docker with the default image and Docker Hub credentials.
Jenkins Pipeline Setup
Create a Multibranch Pipeline:
Go to Dashboard > All > enter your project name and choose 'Multibranch Pipeline' > create.

Webhook URL:

plaintext
Copy code
http://<your-instance-public-ip>:8080/multibranch-webhook-trigger/invoke?token=Devesh121
Jenkinsfile Example:

groovy
Copy code
pipeline {
    agent any

    stages {
        stage('Deploy To Kubernetes') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'EKS-1', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://<eks-cluster-url>']]) {
                    sh "kubectl apply -f deployment-service.yml"
                }
            }
        }

        stage('verify Deployment') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'EKS-1', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', serverUrl: 'https://<eks-cluster-url>']]) {
                    sh "kubectl get svc -n webapps"
                }
            }
        }
    }
}
Deploy Microservices:
Commit changes in the Jenkinsfile of the associated Git repo branches to trigger the pipeline.

Cleanup
To delete the EKS cluster and associated resources:

bash
Copy code
eksctl delete cluster --name EKS-1 --region ap-south-1 --force
eksctl get nodegroup --cluster=EKS-1	
eksctl delete nodegroup --cluster=EKS-1 --name=node2








