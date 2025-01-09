# Run LiveKit with STUNner in AWS Elastic Kubernetes Service

This repo conatains the deployment manifest for a quick start for STUNner in AWS EKS.
We'll run LiveKit and LiveKit Meet as an example WebRTC service.
Read more in [my blog post]().

## 1. Prerequisites
Make sure we have the necessary tools installed and configured. Here’s what you’ll need:
 - AWS CLI: For interacting with AWS services from the command line.
 - `eksctl`: A CLI tool specifically designed for creating and managing EKS clusters.
 - `kubectl`: The Kubernetes CLI for interacting with your cluster.
 - `helm`: A package manager for Kubernetes used to install and manage applications.

Install the AWS CLI:
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
# you'll be promted to give your API key and secret
```


Install `eksctl`:
```
curl -L "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" -o eksctl.tar.gz
tar -xzf eksctl.tar.gz -C /usr/local/bin
rm eksctl.tar.gz
eksctl version
```

Install `kubectl`:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/$(uname -s | tr '[:upper:]' '[:lower:]')/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

Install `helm`:
```
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 2. Setting Up an EKS Cluster

We'll use `eksctl` to deploy an EKS cluster.
It’s a command-line utility originally built by Weaveworks (now fully managed by AWS) that significantly simplifies creating and managing EKS clusters. 
With eksctl, you can define your cluster in a YAML configuration file and create it with a single command.
Check `cluster.yaml` and modify to your own use-case.
After that you can simply create an EKS cluster with the following command (usually takes about 10-15 mins to spun up the whole cluster):
```
eksctl create cluster -f cluster.yaml
```

Check the cluster status:
```
eksctl get cluster
kubectl get nodes
```


You should see output listing the nodes in your cluster, similar to:
```
NAME                                             STATUS   ROLES    AGE     VERSION
ip-192-168-29-7.eu-central-1.compute.internal    Ready    <none>   3h15m   v1.30.7-eks-59bf375
ip-192-168-63-16.eu-central-1.compute.internal   Ready    <none>   3h21m   v1.30.7-eks-59bf375
ip-192-168-67-17.eu-central-1.compute.internal   Ready    <none>   3h21m   v1.30.7-eks-59bf375
```

## 3. Installing the AWS Load Balancer Controller

One of the most crucial steps in setting up your EKS cluster for any publicly reachable application is installing the AWS Load Balancer Controller.
This component can automatically provision and manage AWS Application Load Balancers (ALBs) and Network Load Balancers (NLBs) for Kubernetes services.
To install the AWS Load Balancer Controller you'll need the following steps: 
Follow these steps to set up the AWS Load Balancer Controller.

1. Create an IAM Role for the Load Balancer Controller:
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
POLICY_ARN=$(aws iam list-policies --query 'Policies[?PolicyName==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text)
eksctl create iamserviceaccount \
  --cluster=stunner-livekit \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn="$POLICY_ARN" \
  --approve
```

2. Install the AWS Load Balancer Controller with Helm:
```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=stunner-livekit \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --namespace kube-system \
  --set region=eu-central-1
```

At this point, the AWS Load Balancer Controller is installed and ready to manage traffic for your cluster. This step is critical for enabling STUNner to function in AWS EKS, as it requires the Load Balancer Controller to route traffic to its components.

## 4. Install Nginx Ingress and Cert-Manager for HTTP(S) Traffic and TLS Certificate Management

Although the Nginx Ingress and Cert-Manager aren’t specific to AWS or EKS, they are standard tools for managing ingress resources and TLS certificates in Kubernetes clusters which we’ll need for LiveKit. 

To install Nginx and Cert-Manager simply execute the following:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/aws/deploy.yaml
kubectl annotate service -n ingress-nginx ingress-nginx-controller service.beta.kubernetes.io/aws-load-balancer-scheme=internet-facing
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
# wait for cert-manager to be in ready state before applying the issuer
kubectl wait --for=condition=Ready -n cert-manager pod  -l app.kubernetes.io/component=webhook --timeout=90s
kubectl apply -f manifests/issuer.yaml

kubectl get services -n ingress-nginx
```

After checking the services for the ingress-nginx namespace you should see the public IP (or hostname) that the AWS Load Balancer Controller created for you. This IP you should register in your DNS provider since this will route the HTTP traffic into your cluster.

## 5. Install STUNner

STUNner is a Kubernetes-based media gateway that simplifies the deployment of WebRTC applications in cloud-native environments. It acts as a STUN and TURN server, allowing WebRTC clients to establish peer-to-peer connections even when behind NATs or firewalls.
In this deployment, STUNner serves as a gateway between external WebRTC clients and the LiveKit media server running in Kubernetes. 

To install STUNner run the following:
```
helm repo add stunner https://l7mp.io/stunner
helm repo update
helm install stunner stunner/stunner-gateway-operator \
  --namespace stunner \
  --create-namespace

kubectl apply -f manifests/stunner.yaml
```

There are many STUNner and AWS specific happening here, so be sure to check out the blog for more details.

## 6. Deploying LiveKit

To get LiveKit up and running, we’ll deploy several interconnected components, each with its own Deployment, Service, and Ingress configuration. These components include:
 - `Redis`: Redis serves as a backend for LiveKit, managing session storage and state. It is a lightweight, in-memory database optimized for low-latency operations, making it an ideal choice for real-time communication platforms.
 - `LiveKit Server`: The LiveKit server handles signaling, WebRTC sessions, and media routing. It is the core of the LiveKit architecture and interacts with STUNner to manage media transport for clients.
 - `Demo Token Server`: Authentication in LiveKit requires generating JWT tokens that define user permissions and capabilities for a session. The demo token server is a simple service that generates these tokens based on predefined secrets. It’s a handy utility for testing and demonstration purposes.
 - `LiveKit Meet Demo App`: To showcase the capabilities of LiveKit, we’ll deploy LiveKit Meet, a demo application that provides a fully functional video conferencing interface. This app is an excellent starting point for exploring LiveKit’s features or building your own WebRTC application.

Be sure to check the file `manifest/livekit.yaml` and modify it to your own usecase (especially ingress host names).
After that you can simply run the following to install LiveKit:
```
kubectl create namespace livekit
kubectl apply -f manifests/livekit.yaml -n livekit
kubectl apply -f manifests/livekit-udproute.yaml
```

At this point, by deploying LiveKit alongside Redis, the token server, and the LiveKit Meet app, you’ll have a fully functional WebRTC platform running in Kubernetes, powered by STUNner for efficient and scalable media transport. This architecture not only avoids the pitfalls of host networking but also aligns with Kubernetes best practices, ensuring modularity, security, and cloud-native scalability.

