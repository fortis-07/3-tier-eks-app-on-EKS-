# 📘  3-Tier Application Deployment on Amazon EKS with Ingress and Cloudflare 


---

## 📋 Overview

This guide provides a complete walkthrough for deploying a production-ready 3-tier application on Amazon Elastic Kubernetes Service (EKS). The deployment includes PostgreSQL database connectivity, backend and frontend apps, Ingress setup via AWS Load Balancer Controller, DNS mapping using Cloudflare, and SSL termination.

### 🧠 You Will Learn To:

1. Create an EKS Cluster with `eksctl`
2. Connect to a PostgreSQL database using ExternalName service
3. Configure Kubernetes secrets and config maps
4. Deploy backend and frontend workloads
5. Run migration jobs
6. Install and configure AWS Load Balancer Controller
7. Create and apply Ingress resources
8. Connect a custom domain (`ponmile.com.ng`) via Cloudflare
9. Enable HTTPS/SSL with Cloudflare settings

---

## 🧰 Prerequisites

* AWS CLI configured with proper IAM permissions
* `eksctl`, `kubectl`, `helm` installed
* Docker images for backend/frontend available in a container registry
* Registered domain name (`ponmile.com.ng`) managed in Cloudflare
* Verified public subnets for load balancer creation

---

## 1️⃣ Create the EKS Cluster

```bash
eksctl create cluster \
--name Fortis-v2 \
--region us-east-1 \
--version 1.29 \
--nodegroup-name workers \
--node-type t3.medium \
--nodes 2 \
--nodes-min 1 \
--nodes-max 3 \
--managed
```

<img width="1763" height="701" alt="image" src="https://github.com/user-attachments/assets/3f8b4631-c32c-4de8-b5e2-3f7dd10c8e18" />


This will create a new VPC, subnets, Nat gateway, and an EKS cluster with a managed node group.


*Verify the Nodes cluster Creation*

```bash
aws eks list-clusters
```

<img width="909" height="160" alt="Screenshot 2025-07-25 162140" src="https://github.com/user-attachments/assets/b3853bb6-e1cd-4f7b-8958-43235d3996df" />


You need to run these commands to connect your local machine to your Amazon EKS (Kubernetes) cluster so you can manage it using kubectl.

```bash
export cluster_name=Fortis-v2 # Name of the cluster
```
```echo $cluster_name```

**Configure the kube config**

```bash 
aws eks update-kubeconfig --name $cluster_name --region us-east-1
```

**Get the current context**

```bash
kubectl config current-context
```

<img width="1363" height="124" alt="Screenshot 2025-07-25 165039" src="https://github.com/user-attachments/assets/59a98840-a97c-40a9-8917-2bd1353269e6" />


## 2️⃣ Creating RDS Postgres instance
I will create a PostgreSQL RDS instance in the same VPC as the EKS cluster. I will deploy my RDS instance using the private subnets from the VPC.


**Get the VPC ID associated with your cluster**

```bash
aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 --query "cluster.resourcesVpcConfig.vpcId" --output text
```
<img width="779" height="101" alt="image" src="https://github.com/user-attachments/assets/575d1f02-5b43-4799-a1f1-4a89e130e88f" />

**Get VPC ID**

```bash
VPC_ID=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 --query "cluster.resourcesVpcConfig.vpcId" --output text)
```

**Find private subnets (subnets without a route to an internet gateway)**

```bash
PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[?MapPublicIpOnLaunch==\`false\`].SubnetId" \
  --output text \
  --region eu-west-1)
```

**Create a subnet group using only private subnet**

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name akhilesh-postgres-private-subnet-group \
  --db-subnet-group-description "Private subnet group for Akhilesh PostgreSQL RDS" \
  --subnet-ids subnet-0115609bc602b388e subnet-0611f6ecae28a510c subnet-00a21441953091d88 \
  --region eu-west-1
```

<img width="813" height="770" alt="image" src="https://github.com/user-attachments/assets/7119f52a-327a-432f-935d-b19762f7b6bd" />

**Create security group for RDS**

Create a security group that allows the cluster nodes to reach RDS on the port 5432. For security reasons, we should only open the ports from a particular security group instead of all (0.0.0.0/0)
```bash
aws ec2 create-security-group \
--group-name postgressg \
--description "SG for RDS" \
--vpc-id $VPC_ID \
--region us-east-1
```
## Creating the security group ingress rule to allow inbound on port 5432 from Cluster SG

**Store the security group ID**

```bash
SG_ID=$(aws ec2 describe-security-groups \
--filters "Name=group-name,Values=postgressg" "Name=vpc-id,Values=$VPC_ID" \
--query "SecurityGroups[0].GroupId" \
--output text \
--region us-east-1))
```

**The SG attached with cluster nodes**

```bash
NODE_SG=$(aws eks describe-cluster --name Akhilesh-cluster --region eu-west-1 \
--query "cluster.resourcesVpcConfig.securityGroupIds[0]" --output text)
```

**Allow cluster to reach rds on port 5432**
```bash
aws ec2 authorize-security-group-ingress \
--group-id sg-02b8f5325b7833969 \
--protocol tcp \
--port 5432 \
--source-group $NODE_SG \
--region us-east-1
```

<img width="799" height="761" alt="image" src="https://github.com/user-attachments/assets/28b2815c-72ac-4bb3-8f14-209275425092" />

**Creating an RDS instance that can only be privately accessed.**

```bash
aws rds create-db-instance \
  --db-instance-identifier fortis-postgres \
  --db-instance-class db.t4g.small \
  --engine postgres \
  --engine-version 15 \
  --allocated-storage 20 \
  --master-username postgresadmin \
  --master-user-password YourStrongPassword123! \
  --db-subnet-group-name v2-postgres-private-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --no-publicly-accessible \
  --backup-retention-period 7 \
  --storage-type gp2 \
  --region us-east-1
```

<img width="1908" height="721" alt="Screenshot 2025-07-26 104723" src="https://github.com/user-attachments/assets/5d003bce-1b15-4305-a916-53e3afeb0f9c" />

<img width="795" height="730" alt="image" src="https://github.com/user-attachments/assets/8b1a9d85-5462-4691-8e04-b6bdc56494f9" />


**Clone the repo**
git clone https://github.com/kubernetes-zero-to-hero.git
cd 3-tier-app-eks/k8s
- Create a namespace with the name 3-tier-app-eks

```bash
kubectl apply -f namespace.yaml
```
<img width="794" height="83" alt="image" src="https://github.com/user-attachments/assets/a0e3104a-84c7-486a-bd2a-1411bff932bd" />

**Connect to External PostgreSQL Using ExternalName**

Now we can use this service to point to RDS, if we need to update the cluster, we can update this service and everything will stay the same.

Now we can use the DNS name for RDS using service discovery. it follows the format service_name.namespace.svc.cluster.localWe should be able to use postgres-db.3-tier-app-eks.svc.cluster.local to connect to the database


Apply the manifest:

```bash
kubectl apply -f postgres-service.yaml
```

Confirm DNS resolution:

```bash
kubectl run -it --rm --restart=Never dns-test --image=tutum/dnsutils -- dig postgres-db.3-tier-app-eks.svc.cluster.local
```

<img width="792" height="643" alt="image" src="https://github.com/user-attachments/assets/6aaf0f3a-9b3d-49de-be39-7ed1390a12ef" />

(PS: Keep the Postrgress security group opened if you experienced issues connecting to the postgres server.)



## 3️⃣ Configure Secrets and ConfigMap

Base64 encode credentials:

```bash
echo 'postgresadmin' | base64
echo 'YourStrongPassword123!' | base64
echo 'postgresql://postgresadmin:YourStrongPassword123!@postgres-db.3-tier-app-eks.svc.cluster.local:5432/postgres' | base64
```

Use them in `secrets.yaml` and `configmap.yaml` and apply:

```bash
kubectl apply -f secrets.yaml
kubectl apply -f configmap.yaml
```

## 4️⃣ Deploy Backend and Frontend

**Apply workloads:**

```bash
kubectl apply -f backend-deployment.yaml
kubectl apply -f frontend-deployment.yaml
```

<img width="1468" height="158" alt="image" src="https://github.com/user-attachments/assets/72dbfa5e-b1b9-460f-8848-0d973e80b741" />

**Checking the deployments and services**

```bash
kubectl get deployment -n 3-tier-app-eks
```
<img width="786" height="169" alt="image" src="https://github.com/user-attachments/assets/a0330eeb-2fd4-4b68-bc6c-deb55d3d57b5" />


```bash
kubectl get svc -n 3-tier-app-eks
```

<img width="1503" height="38" alt="image" src="https://github.com/user-attachments/assets/1ea0a8c7-3e1a-4ec7-a061-9dce2511a632" />

```bash
kubectl get po -n 3-tier-app-eks
```
<img width="1606" height="333" alt="image" src="https://github.com/user-attachments/assets/43eec834-da77-4fc3-b915-64dd47eb0d9f" />


## 5️⃣ Run Database Migration Job

Apply the `migration-job.yaml`:

```bash
kubectl apply -f migration-job.yaml
```

<img width="1503" height="69" alt="image" src="https://github.com/user-attachments/assets/6c8cea03-fb3d-4d38-83f0-10c7ea8920af" />

Verify completion:

```bash
kubectl get job -A
```

<img width="1343" height="80" alt="image" src="https://github.com/user-attachments/assets/d3dd22d2-4860-48de-871d-b1f8897078ad" />

Troubleshoot (if necessary):

```bash
kubectl logs job/database-migration -n 3-tier-app-eks
```

<img width="1721" height="720" alt="image" src="https://github.com/user-attachments/assets/afd050f5-0461-448b-b42e-cb06f56564fb" />



## 6️⃣ Install AWS Load Balancer Controller

### a. Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster Fortis-v2 --approve
```

<img width="1761" height="98" alt="image" src="https://github.com/user-attachments/assets/87a3c0b3-1e30-4c37-86ad-02b736f39647" />


### b. Create IAM Policy and Service Account

```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json
```

<img width="1754" height="306" alt="image" src="https://github.com/user-attachments/assets/96c07bae-14dc-4971-b447-258e7a89fdde" />


<img width="1615" height="440" alt="image" src="https://github.com/user-attachments/assets/35f9eeb3-cbc6-4374-a24a-fb6b8a8c8f97" />



```bash
eksctl create iamserviceaccount \
--cluster=Fortis-v2 \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```

<img width="1654" height="340" alt="image" src="https://github.com/user-attachments/assets/09e697e2-aca8-45a2-821f-1358ab9a821d" />


### c. Install Using Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system \
--set clusterName=Fortis-v2 \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=us-east-1 \
--set vpcId=<your-vpc-id>
```

<img width="1703" height="260" alt="image" src="https://github.com/user-attachments/assets/b4b59eb5-d165-4bb6-a210-be39d672c538" />

##  7️⃣ Tag Public Subnets for ALB

```bash
aws ec2 create-tags --resources subnet-02e3680c7a052c52f subnet-0ea63a3626c8544ce \
--tags Key=kubernetes.io/role/elb,Value=1
```

<img width="1754" height="647" alt="image" src="https://github.com/user-attachments/assets/dc04cff0-4ed4-4e3d-93df-4a3bcaff5550" />



## 8️⃣Deploy Ingress Resource

Create `ingress.yaml`

Apply the manifest:

```bash
kubectl apply -f ingress.yaml
```

Verify the resource creation:

```bash
kubectl get ingress -n 3-tier-app-eks
```

<img width="1736" height="663" alt="image" src="https://github.com/user-attachments/assets/26e9fa06-3487-4a88-966c-f1202eb09ff8" />

Ingress status shows reconciled, which means it has created the load balancer. 

You can go to your AWS console where you will see an ALB there. Copy the DNS name for it and paste it into the browser, and you should be able to see your app there.


<img width="1881" height="836" alt="Screenshot 2025-07-24 220301" src="https://github.com/user-attachments/assets/db88db2a-adf7-41f5-afde-06f00ceb62b5" />


## 9️⃣ Point Domain to ALB via Cloudflare

**Add Your Domain to Cloudflare**

- Go to your Cloudflare dashboard.
- Click "Add a Site" → enter ponmile.com.ng.
- Cloudflare will scan your current DNS records.
- Update nameservers at your domain registrar (e.g., Qservers, GoDaddy, Namecheap) to point to the ones Cloudflare gives you.

**Create a DNS Record in Cloudflare**

 Log in to Cloudflare dashboard
- Navigate to **DNS > Records**
- Add a new record:

  * **Type:** CNAME
  * **Name:** `ponmile.com.ng`
  * **Target:** ALB DNS name from Ingress

<img width="1398" height="561" alt="Screenshot 2025-07-25 140941" src="https://github.com/user-attachments/assets/9bfbe185-01c9-468e-9d53-d7697bf71318" />

## ✅ 11️⃣ Verify Access

### a. Browser Access:

* Open [https://ponmile.com.ng](https://ponmile.com.ng)


<img width="1907" height="900" alt="Screenshot 2025-07-25 141441" src="https://github.com/user-attachments/assets/03600d11-3484-4c3a-b786-85a5c00140da" />


## 🎉 Final Notes

Congratulations! You have successfully:

* Provisioned an EKS cluster
* Connected to an external PostgreSQL instance
* Deployed a backend and frontend
* Configured a migration job
* Set up Ingress with AWS ALB
* Mapped a domain name to your app using Clouflare


Your 3-tier app is now live and secure using modern cloud-native architecture.
