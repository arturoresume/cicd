# File: CICD.txt
Author: Arturo Gomez

# References
1. AWS Official Documentation
2. How to setup kubernetes jenkins pipeline on AWS: https://jhooq.com/aws-kubernetes-jenkins-pipeline/

# Overview
The present document contains a summary of the installation of a continuous integration and continuous development environment on AWS, using Docker, Jenkins and Kubernetes in 2022.
One possible consideration and improvement in terms of simplicity and security on the architecture would be using JenkinsX and EKS, without docker. JenkinsX has support for native CI/CD on Kubernetes via Tekton.

# PreRequisites
- AWS account
- AWS CLI for automation via script

# Step By Step

## 1. Start an EC2 instance via AWS console. (Ubuntu 20.04 t2.micro Free Tier)
* Is it important to setup a VPC on AWS with the corresponding security rules and access to specific ports. Using IAM users is also recommended as a best practice on AWS.

## 2. SSH on the instance via SSH Key Pair. Install JDK, Jenkins, eksctl, kubectl
$ ssh -i "***.pem" ubuntu@[IP]
$ sudo apt-get update  
$ sudo apt install openjdk-11-jre-headless
$ java --version
$ wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
$ sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'  
$ sudo apt-get install jenkins
$ sudo service jenkins status 
$ sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Setup Jenkins
http://[EC2_IP_ADDRESS]:8080/login
- Create Admin Jenkins User
- Install tools and software required for building the app. Eg. Ruby on Rails, or Python, or Gradle, etc...

## 3. Install Docker
$ sudo su - jenkins  
$ sudo apt install docker.io
$ sudo usermod -aG docker jenkins 

## 4. Install AWS CLI
$ sudo apt install awscli 
$ aws --version 
$ aws configure

## 5. Install Kubectl
$ curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod +x ./kubectl 
$ sudo mv ./kubectl /usr/local/bin
$ kubectl version  --output=yaml

## 6. Install eksctl and setup EKS Cluster
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin 
$ eksctl version
$ eksctl create cluster --name eks-cluster --version 1.22 --region us-east-1 --nodegroup-name worker-nodes --node-type t3.small --nodes 2
* Verify via https://us-east-1.console.aws.amazon.com/eks/home?region=us-east-1#

## 7. Add Docker and GitHub Credentials into Jenkins
Goto -> Jenkins -> Manage Jenkins -> Manage Credentials -> Stored scoped to jenkins -> global -> Add Credentials

## 8. Setup CI/CD Stages

Jenkins Stage-1: Replace github URL with Repo URL
     stage("Git Clone"){
        git credentialsId: 'GIT_HUB_CREDENTIALS', url: 'https://github.com/jonyhub/apiSampleJava'
    }

Jenkins Stage-2: Build
stage('Gradle Build') {
    sh './gradlew build'   # For Java App, Or analogous command for Ruby on Rails, Python , Node JS, etc
}

Jenkins Stage-2.1: Tests
stage('Run Tests') {
    sh 'bin/rails test'   # For Java App, Or analogous command for Ruby on Rails, Python , Node JS, etc
}

# If tests are succesful continue. Else Notify.
Jenkins stage-3 : Create Docker Container and push to Docker Hub
stage("Docker build"){
    sh 'docker version'
    sh 'docker build -t mv-docker'
    sh 'docker image list'
    sh 'docker tag mv-docker mv-docker/mv-docker:mv-docker'
}
stage("Push Image to Docker Hub"){
        sh 'docker push  mv-docker/mv-docker:mv-docker'
}

Jenkins stage-4 : Kubernetes deployment
stage("kubernetes deployment"){
  sh 'kubectl apply -f deployment.yml'
}

## 9. Setup CI/CD Pipeline
- Goto Jenkins-> New Item -> Pipeline -> Pipeline Tab

node {
    stage("Git Clone"){
        git credentialsId: 'GIT_HUB_CREDENTIALS', url: 'https://github.com/jonyhub/apiSampleJava'
    }
    stage('Gradle Build') {
       sh './gradlew build'
    }
    stage('Run Tests') {
        sh 'bin/rails test'   # For Java App, Or analogous command for Ruby on Rails, Python , Node JS, etc
    }

    stage("Docker build"){
        sh 'docker version'
        sh 'docker build -t mv-docker'
        sh 'docker image list'
        sh 'docker tag mv-docker mv-docker/mv-docker:mv-docker'
    }

    withCredentials([string(credentialsId: 'DOCKER_HUB_PASSWORD', variable: 'PASSWORD')]) {
        sh 'docker login -u $MY_DOCKER_HUB_USER -p $PASSWORD'
    }

    stage("Push Image to Docker Hub"){
        sh 'docker push  mv-docker/mv-docker:mv-docker'
    }
    
    stage("kubernetes deployment"){
        sh 'kubectl apply -f deployment.yml'
    }
} 

## 10. Run Pipeline

Goto -> Pipeline Name -> Build Now

$ kubectl get deployments
$ kubectl get service
