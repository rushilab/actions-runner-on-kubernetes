# Create a new EKS cluster using eksctl

This document provides step-by-step guide on how to create new eks cluster using eksctl

## Pre-requisites:
* AWS account with valid credentials / permissions
* Client machine must have following tools installed:
  - Kubectl
  ```
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  sudo install -o root -g root -m 0755 kubectl /usr/local/sbin/kubectl
  ```
  - eksctl
  ```
  curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp \
  sudo mv /tmp/eksctl /usr/local/bin
  ```

## Prepare:
* Edit the configuration files:
  * Simple deploy: `eksctl/eksctl-simple.yaml`
  * Using existing VPC: `eksctl/eksctl-existing-vpc.yaml`

## Create EKS cluster:
* Run below command to create a simple EKS cluster
```
eksctl create cluster -f eksctl/eksctl-simple.yaml
```
* Run below command to create a EKS cluster with existing VPC
```
eksctl create cluster -f eksctl/eksctl-existing-vpc.yaml
```

## Delete EKS cluster:
* Run below command to delete a simple EKS cluster
```
eksctl delete cluster -f eksctl/eksctl-simple.yaml
```
* Run below command to delete a EKS cluster with existing VPC
```
eksctl delete cluster -f eksctl/eksctl-existing-vpc.yaml
```
