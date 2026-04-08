**🚀 EKS Monitoring Setup with Prometheus & Grafana**

This project demonstrates a complete end-to-end monitoring setup on AWS EKS using Prometheus and Grafana deployed via Helm.

**📌 Architecture Overview**

EKS Cluster

    Prometheus (Metrics Collection)
    Grafana (Visualization)
    Metrics Server (Resource Metrics)
   
**🛠️ Prerequisites**

AWS Account
EC2 Instance (Ubuntu - t2.medium recommended)
IAM Role with required permissions

**⚙️ Setup Steps**

**1️⃣ Install AWS CLI**

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

sudo apt install unzip -y

unzip awscliv2.zip

sudo ./aws/install

**2️⃣ Install kubectl**

curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x kubectl

sudo mv kubectl /usr/local/bin

kubectl version 

**3️⃣ Install eksctl**

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version

**4️⃣ Install Helm**

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

helm version

**5️⃣ Create EKS Cluster**

eksctl create cluster \
--name eks2 \
--version 1.24 \
--region us-east-1 \
--nodegroup-name worker-nodes \
--node-type t2.medium \
--nodes 2 \
--nodes-min 2 \
--nodes-max 3

**6️⃣ Update kubeconfig (if needed)**

aws eks update-kubeconfig --region us-east-1 --name eks2

**7️⃣ Install Metrics Server**

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl get deployment metrics-server -n kube-system

**📊 Prometheus Setup**
**Add Helm Repo**

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

**Create Namespace**

kubectl create namespace prometheus

Install Prometheus

helm install prometheus prometheus-community/prometheus \
--namespace prometheus \
--set alertmanager.persistentVolume.storageClass="gp2" \
--set server.persistentVolume.storageClass="gp2"


**🔐 Configure OIDC Provider**

oidc_id=$(aws eks describe-cluster --name eks2 --region us-east-1 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)


aws iam list-open-id-connect-providers | grep $oidc_id


eksctl utils associate-iam-oidc-provider \
--cluster eks2 \
--approve \
--region ap-south-1


**💾 Install EBS CSI Driver**
**Create IAM Service Account**

eksctl create iamserviceaccount \
--name ebs-csi-controller-sa \
--namespace kube-system \
--cluster eks2 \
--attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
--approve \
--role-only \
--role-name AmazonEKS_EBS_CSI_DriverRole \
--region ap-south-1


**Attach Addon**

eksctl create addon \
--name aws-ebs-csi-driver \
--cluster eks2 \
--service-account-role-arn arn:aws:iam::<ACCOUNT-ID>:role/AmazonEKS_EBS_CSI_DriverRole \
--force \
--region ap-south-1


**📈 Access Prometheus**

kubectl get pods -n prometheus
kubectl port-forward deployment/prometheus-server 9090:9090 -n prometheus

👉 Open: http://localhost:9090

**📊 Grafana Setup**
**Add Helm Repo**

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

**Create Namespace**

kubectl create namespace grafana

**Install Grafana**

helm install grafana grafana/grafana \
--namespace grafana \
--set persistence.storageClassName="gp2" \
--set persistence.enabled=true \
--set adminPassword='EKS!sAWSome' \
--set service.type=LoadBalancer

**🔍 Verify Grafana**

kubectl get pods -n grafana

kubectl get svc -n grafana

**👉 Copy EXTERNAL-IP and access Grafana in the browser**

--------------------------------------------------------------------------------------------------------------------------------------------------------------------

**🧠 Key Learnings**

Prometheus uses pull-based monitoring

Difference between Metrics Server vs Prometheus

Namespace isolation for monitoring stack

Dynamic provisioning using EBS CSI Driver

Kubernetes internal DNS (svc.cluster.local)

----------------------------------------------------------------------------------------------------------------------------------------------------------------------


**🧹 Cleanup**
eksctl delete cluster --name eks2 --region ap-south-1

**📌 Conclusion**

This setup provides a production-like monitoring stack for Kubernetes workloads using Prometheus and Grafana, enabling real-time observability and performance tracking.



----------------------------------------------------------------------------------------------------------------------------------------------------------------


**📖 Project Explanation**

This project demonstrates a production-like monitoring setup on AWS EKS using Prometheus and Grafana, deployed via Helm.

The goal of this project is to achieve real-time observability into Kubernetes infrastructure and workloads by collecting, storing, and visualizing metrics.

**🔹 How it works**

An Amazon EKS cluster is provisioned using eksctl with autoscaling worker nodes.
Kubernetes Metrics Server is installed to provide real-time CPU and memory usage of pods and nodes.
Prometheus is deployed using Helm to collect and store time-series metrics from the cluster.
Persistent storage is enabled using AWS EBS (gp2) via the EBS CSI driver to ensure data durability.
Grafana is deployed and exposed via a LoadBalancer to provide visual dashboards.

Grafana is connected to Prometheus using Kubernetes internal service discovery:

http://prometheus-server.prometheus.svc.cluster.local

Dashboards are used to visualize metrics such as CPU usage, memory usage, and cluster health.


**🧠 Key Concepts Explained**

**🔹 Amazon EKS (Elastic Kubernetes Service)**

Managed Kubernetes service by AWS used to run containerized applications without managing control plane infrastructure.

**🔹 kubectl**

Command-line tool used to interact with Kubernetes clusters (create, update, delete resources).

**🔹 eksctl**

CLI tool used to simplify EKS cluster creation and management.

**🔹 Helm**

Package manager for Kubernetes used to deploy applications using reusable charts.


**🔹 OIDC (OpenID Connect) in EKS**

OIDC (OpenID Connect) is used in Amazon EKS to enable secure authentication between Kubernetes service accounts and AWS IAM roles.

   **🔸 Why OIDC is required?**

By default, Kubernetes pods do not have permission to access AWS services (like EBS, S3, etc.).

Instead of hardcoding AWS credentials inside pods (which is insecure), we use:

**👉 IAM Roles for Service Accounts (IRSA)**

**🔸 How it works in this project**

An OIDC provider is associated with the EKS cluster
A Kubernetes service account is created
An AWS IAM role is attached to that service account
Pods using that service account automatically get AWS permissions

**🔸 In this setup**

**OIDC is used to enable the EBS CSI Driver to:**

Dynamically create EBS volumes
Attach volumes to worker nodes
Manage persistent storage securely

**🔹 Prometheus**

Monitoring tool that:

Collects metrics using a pull-based model
Stores data as time-series
Scrapes metrics from Kubernetes services and pods


**🔹 Grafana**

Visualization tool used to:

Create dashboards
Query Prometheus
Display metrics in graphs and charts

**🔹 Metrics Server**

Collects short-term resource usage (CPU/memory) for Kubernetes components.

Used for:

kubectl top pods
kubectl top nodes


**🔹 Namespace**

Logical separation in Kubernetes.

Used in this project:

prometheus → monitoring backend
grafana → visualization layer

**🔹 Persistent Volume (PV) & Persistent Volume Claim (PVC)**

Used to provide persistent storage for applications.

PVC → request for storage
PV → actual storage resource


**🔹 EBS CSI Driver**

Enables Kubernetes to dynamically create and attach AWS EBS volumes for persistent storage.

**🔹 StorageClass (gp2)**

Defines the type of storage (AWS EBS in this case) used for dynamic provisioning.

**🔹 Service Discovery (svc.cluster.local)**

Kubernetes internal DNS is used for communication between services.

Example:

prometheus-server.prometheus.svc.cluster.local

**🔹 Port Forwarding**

Used to access internal Kubernetes services locally:

kubectl port-forward deployment/prometheus-server 9090:9090 -n prometheus


-------------------------------------------------------------------------------------------------------------------------------------------------------------------


**🔄 End-to-End Flow**

Kubernetes Cluster
        ↓
Prometheus (scrapes metrics)
        ↓
Time-Series Database
        ↓
Grafana (queries Prometheus)
        ↓
Dashboards & Visualization


----------------------------------------------------------------------------------------------------------------------------------------------------------------

**🎯 Outcome**
Achieved real-time monitoring of Kubernetes workloads
Enabled persistent storage for metrics
Visualized system health using Grafana dashboards
Improved understanding of production-grade observability
