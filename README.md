# Self-Hosted Github Actions runner on Kubernetes Cluster

This document provides step-by-step guide on how to deploy self-hosted Github Actions Runner within EKS cluster with Autoscaling enabled via Webhooks.

Source / Referance -- https://github.com/actions-runner-controller/actions-runner-controller

Index:
- [Pre-requisites](#pre-requisites)
- [Prepare](#prepare)
- [Start deploying Actions Runner Controller](#start-deploying-actions-runner-controller)
  - [Install cert-manager](#install-cert-manager)
  - [Install runner controller](#install-runner-controller)
  - [Enable Webhook](#enable-webhook)
- [Deploy self-hosted runner](#deploy-self-hosted-runner)  
- [Setup Github organization / repository with Webhook](#setup-webhook-in-github)  
- [Test / verify](#test-verify)  


## Pre-requisites:
* A working EKS cluster
  - If you don't have or want to create one for testing, use this guideline to quickly create EKS cluster -- [Create EKS Cluster](./eksctl/docs/eksctl-create-eks-cluster.md)
* Github account
* The Github Organization / repository where the runners needs to be registered
* Client machine must have following tools installed:
  - Kubectl
  ```
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  sudo install -o root -g root -m 0755 kubectl /usr/local/sbin/kubectl
  ```
  - Helm
  ```
  curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
  chmod 700 get_helm.sh
  ./get_helm.sh
  ```
* `~/.kube/config` must have valid configuration exist


## Prepare:
* Github authentication: We can use App authentication, but for our demo purpose we will use Personal Access Token (PAT)
  * Follow the guideline here to Create a PAT with appropriate permissions -- https://github.com/actions-runner-controller/actions-runner-controller#deploying-using-pat-authentication
  * Note down the token for latter use.

* Create the `actions-runner-controller/custom-values.yaml`
```  
cp actions-runner-controller/custom-values.tpl actions-runner-controller/custom-values.yaml
```
* Edit the file and replace the value of `github_token:` with the valid token you created before
> NOTE: Since this is a secret, `actions-runner-controller/custom-values.yaml` is added into `.gitignore`


## Start deploying Actions Runner Controller:

### Install cert-manager:
* Add and update the helm repository
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
```
* Install chart.
```
helm install --wait --create-namespace --namespace cert-manager \
cert-manager jetstack/cert-manager --version v1.3.0 --set installCRDs=true
```
* Verify installation
```
kubectl --namespace cert-manager get all
```

### Install runner controller:
* Add and update the helm repository
```
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update
```
* Install chart
```
helm install -f actions-runner-controller/custom-values.yaml --wait --namespace \
actions-runner-system --create-namespace \
actions-runner-controller \
actions-runner-controller/actions-runner-controller
```
* Verify installation
```
kubectl --namespace actions-runner-system get all
```

### Enable Webhook:
Webhook can be exposed using various ways like Using `LoadBalancer`, `IngressController`, `NodePort`, etc.
However, for demo purpose we are using type LoadBalancer, this will expose the webhook with classic ELB in AWS on port TCP 80

* Enable Webhook -- Type LoadBalancer
```
helm upgrade --install --namespace \
actions-runner-system --create-namespace --wait \
actions-runner-controller \
actions-runner-controller/actions-runner-controller --set \
"githubWebhookServer.enabled=true,githubWebhookServer.service.type=LoadBalancer"
```
* Verify: You should be able to see the newly created classic LoadBalancer. Note down its DNS name for future referance at step [Setup Github organization / repository with Webhook](#setup-webhook-in-github)

## Deploy self-hosted runner:
* Create Namespace for self-hosted runners
```
kubectl create namespace self-hosted-runners
```
* Update the value of `organization:` or `repository:` within `actions-runner-controller/runners-deployment.yaml`.
* Deploy Autoscaling and Webhook enabled runners
```
kubectl --namespace self-hosted-runners apply -f actions-runner-controller/runners-deployment.yaml
```
* Verify
```
kubectl --namespace self-hosted-runners get runner
```
```
kubectl --namespace self-hosted-runners get pods
```

> Notes:
> * We have configured runner deployment with min/max replicas to 0/4 respectively. Hence initially, you won't be able to see any runners / pods running.
> * The runners will be created via Webhook trigger.

## Setup Github organization / repository with Webhook:
* In your Github account, navigate to your organization / repository where you want to register Runners
* Go to Settings --> Webhooks --> Add Webhook
* Fill in the following details
  * Payload URL: HTTP URL of the Load Balancer created during [Enable Webhook](#enable-webhook) step
  * Content type: application/json
  * Which events would you like to trigger this webhook?: select the option `Let me select individual events.` and select following permissions:
    * Workflow jobs
    * Pushes
  * Make sure to select `Active`
  * Click on Add Webhook to save the changes

## Test and verify the runners:
* Copy the `workflow_samples/workflows` directory under `.github` folder within your repository
* In a new terminal, start continuously watching the runners within eks cluster
```
watch kubectl get pods --namespace self-hosted-runners
```
* Push the workflows into your repository
* Once the workflow jobs are executing, you would see the runners / pods are dynamically creating and terminating within the autoscaling limit
* See it in action
