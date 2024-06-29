# Microservice Deployment
## Overview:
This repository contains instructions for deploying a set of microservices using AWS EKS and Jenkins. The deployment involves creating an EC2 instance, setting up an EKS cluster, configuring Jenkins, and deploying the microservices.

## Microservices
There are 11 microservices located in different Git branches. We use a multibranch pipeline and webhook for deployment.

## Inbound Port Configuration
Ensure the following ports are open:
25, 3000-10000, 80, 443, 22, 6443, 465, 27017, 30000-32767 for IPv4 everywhere.

## EC2 Instance Setup
1. **Instance Type:** t2.large (2 CPUs, 8GB RAM),
2. **Key Pair:** Use microservice or any as per choice,
3. **Storage:** 25GB,

## IAM User Creation
Create an IAM user named "EKS user" with the following access keys and policies and save in a file.
- User: dee-microservice
- Access Key: your-access-key
- Secret Access Key: your-secret-access-key

**Policies to Attach**
- AmazonEC2FullAccess
- AmazonEKS_CNI_Policy
- AmazonEKSClusterPolicy
- AmazonEKSWorkerNodePolicy
- AWSCloudFormationFullAccess
- IAMFullAccess
- **Custom Policy:** create a custom policy by this below json data;
```json
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
```
## CLI Tools Installation
Install the following CLI tools on the EC2 instance:

1. **AWS CLI:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
```

2. **kubectl:**
```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```
3. **eksctl:**
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
## EKS Cluster Setup
Configure AWS CLI with your credentials and region (ap-south-1):
```bash
aws configure
```

**Create EKS Cluster**
```bash
eksctl create cluster --name=EKS-1 \
                      --region=ap-south-1 \
                      --zones=ap-south-1a,ap-south-1b \
                      --without-nodegroup

```
**To delete the cluster:**
```bash
eksctl delete cluster --name EKS-1 --region ap-south-1
```

## Associate IAM OIDC Provider to accept the IAM credential in EKS cluster: 
```bash
eksctl utils associate-iam-oidc-provider \
    --region ap-south-1 \
    --cluster EKS-1 \
    --approve
```

## Create EKS Node Group as worker
```bash
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
```

**To delete node groups:**
```bash
eksctl get nodegroup --cluster=EKS-1	
eksctl delete nodegroup --cluster=EKS-1 --name=node2
```
## Jenkins Setup:
**Install Docker:**
```bash
sudo apt install docker.io
sudo chmod 666 /var/run/docker.sock
sudo usermod -aG docker $USER
sudo usermod -aG docker jenkins
```
**Install Java JDK 17:**
```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre
java -version
```
**Install Jenkins:**
```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```
**Enable and Start Jenkins service:**
``` bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
**Access Jenkins:**
Access the jenkins server on port 8080 of your host machine with there public-ip associated:
```bash
"http://<your-instance-public-ip>:8080" 
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Install Plugins in jenkins manage plugins:
- Docker, Docker Pipeline
- Kubernetes, Kubernetes CLI
- Multibranch Scan Webhook Trigger
  
**Configure Plugins:**
- Configure Docker with the default image and Docker Hub credentials.

**Jenkins Pipeline Setup:**
- Create a Multibranch Pipeline:
- Go to Dashboard > All > enter your project name and choose 'Multibranch Pipeline' > create.

Webhook URL:
```plaintext
http://<your-instance-public-ip>:8080/multibranch-webhook-trigger/invoke?token=your-token-name
```
**Jenkinsfile Example:**
```groovy
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
```
## Deploy Microservices:
- Commit changes in the Jenkinsfile of the associated Git repo branches to automatically trigger the pipeline by webhook and jenkins.
- **For security purpose we are going to use a Service Account ,Role and Role binding with service account. for that:**
- **Create a namespace for app deployment isolation.**
  ```bash
  kubectl create namespace webapps
  ```
- **Service account:** create a file with the name of "service-account.yml" and paste below lines.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```
-  apply the service-account.yml file
- **Role:** now create a role with 'vi service-role.yml'.
 ```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
-  apply the service-account.yml file
- **ROLE-Binding:** bind the service-account with the service-role.create a bind file 'account-role-bind.yml';
 ``` yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: app-rolebinding
      namespace: webapps 
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: Role
      name: app-role 
    subjects:
    - namespace: webapps 
      kind: ServiceAccount
      name: jenkins           
  ```
## After role get binded with service account we need a token to communicate with eks cluster:
**Secret:** create a one more 'secret.yml' file and create  the Token.
  ``` yaml
   apiVersion: v1
   kind: Secret
   type: kubernetes.io/service-account-token
   metadata:
     name: mysecretname
     annotations:
       kubernetes.io/service-account.name: myserviceaccount
```
- apply this secret file using kubectl command;
- **Token:**
 ```bash
kubectl describe secret mysecretname -n webapps
```
- **Output:**
```bash
Name:         mysecretname
Namespace:    webapps
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: jenkins
              kubernetes.io/service-account.uid: ac862a73-9a76-4767-93d5-414b96292570

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1107 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IklvT1UzUlFNcnY1aHhGN3ZwMHU5OEhUR3JGOG9XanplQlZxbFBMSVppVUkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJhcHBzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im15c2VjcmV0bmFtZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWM4NjJhNzMtOWE3Ni00NzY3LTkzZDUtNDE0Yjk2MjkyNTcwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OndlYmFwcHM6amVua2lucyJ9.lrsHTx6nWoRatNjHSivhV0LpSOCdhPlpi3hwih4Ys3B5tOseeEr-bjmybxrTRnQb4b1Oe7wrquWqnuXFoVzVrLjk61pZGW66eFQEMcdpEhNSAWN4DyYIArxyw9TIWGtYS3PI9NeyvXCgmR8CPgHkFqiu9ScLInWzVH1QOVt65-TRJmaMyUp_sK-fOIQgdVUBZUqdPnQb4YCilLK4sYy5Yqv-VFvlBwCzhJuy1R3KY8XK19zVvYL-ATr3OOs3nyLUIcLPVXI6fnr6zTK2_Fku7VslrPyJ1DKMOgrWfXzK9guPyjcdMg2-06gLjylyt-a2zQp8VQbxSYVUZT3QhFDBVA
```
- COPY the secret token and store in good place look like this:
``` json
eyJhbGciOiJSUzI1NiIsImtpZCI6IklvT1UzUlFNcnY1aHhGN3ZwMHU5OEhUR3JGOG9XanplQlZxbFBMSVppVUkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJhcHBzIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im15c2VjcmV0bmFtZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWM4NjJhNzMtOWE3Ni00NzY3LTkzZDUtNDE0Yjk2MjkyNTcwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OndlYmFwcHM6amVua2lucyJ9.lrsHTx6nWoRatNjHSivhV0LpSOCdhPlpi3hwih4Ys3B5tOseeEr-bjmybxrTRnQb4b1Oe7wrquWqnuXFoVzVrLjk61pZGW66eFQEMcdpEhNSAWN4DyYIArxyw9TIWGtYS3PI9NeyvXCgmR8CPgHkFqiu9ScLInWzVH1QOVt65-TRJmaMyUp_sK-fOIQgdVUBZUqdPnQb4YCilLK4sYy5Yqv-VFvlBwCzhJuy1R3KY8XK19zVvYL-ATr3OOs3nyLUIcLPVXI6fnr6zTK2_Fku7VslrPyJ1DKMOgrWfXzK9guPyjcdMg2-06gLjylyt-a2zQp8VQbxSYVUZT3QhFDBVA
```
**Credential:** save this credential as plain secret-text in jenkins credentials;
## Now run the deployment file by Jenkinsfile present on main branch. after this we can see the application running on the loadblacer external URL;
```json
5000/TCP       4m27s
frontend                NodePort       10.100.101.12    <none>                                                                     80:31636/TCP   4m24s
frontend-external       LoadBalancer   10.100.78.172    a52df5946b8144ab587cff33845b4104-1447625668.ap-south-1.elb.amazonaws.com   80:32631/TCP   4m23s

// URL: a52df5946b8144ab587cff33845b4104-1447625668.ap-south-1.elb.amazonaws.com
```

**Cleanup:** perform the cleaning task after all finished work;
- To delete the EKS cluster and associated resources:
```bash
eksctl delete cluster --name EKS-1 --region ap-south-1 --force
eksctl get nodegroup --cluster=EKS-1	
eksctl delete nodegroup --cluster=EKS-1 --name=node2
```







