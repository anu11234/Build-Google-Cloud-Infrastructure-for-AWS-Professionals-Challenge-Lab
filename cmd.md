# Build Google Cloud Infrastructure for AWS Professionals: Challenge Lab || **GSP511**

**Command:**

```bash
#!/bin/bash
set -e

echo "========================================================"
echo "    GSP511 CHALLENGE LAB INTERACTIVE SETUP SCRIPT"
echo "========================================================"
echo ""

# ------------------------------------------------------
# Interactive Input Section
# ------------------------------------------------------
read -p "Enter Lab Region (e.g., us-central1): " INPUT_REGION
read -p "Enter Lab Zone (e.g., us-central1-a): " INPUT_ZONE
read -p "Enter User 2 Email (from Lab Credentials panel): " INPUT_USER2

export REGION=${INPUT_REGION:-"us-central1"}
export ZONE=${INPUT_ZONE:-"us-central1-a"}
export USER2_EMAIL=$INPUT_USER2
export PROJECT_ID=$(gcloud config get-value project)

gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

echo ""
echo "--------------------------------------------------------"
echo " Configuration Saved:"
echo " Project ID : $PROJECT_ID"
echo " Region     : $REGION"
echo " Zone       : $ZONE"
echo " User 2     : $USER2_EMAIL"
echo "--------------------------------------------------------"
read -p "Press [ENTER] to start executing all tasks..."

# ==========================================
# TASK 1: Create griffin-dev-vpc & subnets
# ==========================================
echo -e "\n[Task 1] Creating griffin-dev-vpc and subnets..."
gcloud compute networks create griffin-dev-vpc --subnet-mode=custom

gcloud compute networks subnets create griffin-dev-wp \
    --network=griffin-dev-vpc \
    --region=$REGION \
    --range=192.168.16.0/20

gcloud compute networks subnets create griffin-dev-mgmt \
    --network=griffin-dev-vpc \
    --region=$REGION \
    --range=192.168.32.0/20

# ==========================================
# TASK 2: Create griffin-prod-vpc & subnets
# ==========================================
echo -e "\n[Task 2] Creating griffin-prod-vpc and subnets..."
gcloud compute networks create griffin-prod-vpc --subnet-mode=custom

gcloud compute networks subnets create griffin-prod-wp \
    --network=griffin-prod-vpc \
    --region=$REGION \
    --range=192.168.48.0/20

gcloud compute networks subnets create griffin-prod-mgmt \
    --network=griffin-prod-vpc \
    --region=$REGION \
    --range=192.168.64.0/20

# ==========================================
# TASK 3: Create Bastion Host
# ==========================================
echo -e "\n[Task 3] Creating Firewall rules and Bastion Host..."
gcloud compute firewall-rules create griffin-dev-allow-ssh \
    --network=griffin-dev-vpc \
    --allow=tcp:22

gcloud compute firewall-rules create griffin-prod-allow-ssh \
    --network=griffin-prod-vpc \
    --allow=tcp:22

gcloud compute instances create bastion-host \
    --zone=$ZONE \
    --machine-type=e2-medium \
    --network-interface=subnet=griffin-dev-mgmt \
    --network-interface=subnet=griffin-prod-mgmt

# ==========================================
# TASK 4: Create Cloud SQL Instance & Database
# ==========================================
echo -e "\n[Task 4] Creating Cloud SQL Instance (takes 3-5 mins)..."
gcloud sql instances create griffin-dev-db \
    --tier=db-custom-1-3840 \
    --region=$REGION \
    --database-version=MYSQL_5_7

gcloud sql users set-password root --host=% --instance=griffin-dev-db --password=password
gcloud sql databases create wordpress --instance=griffin-dev-db
gcloud sql users create wp_user --instance=griffin-dev-db --password=stormwind_rules --host='%'

# ==========================================
# TASK 5: Create GKE Cluster
# ==========================================
echo -e "\n[Task 5] Creating GKE Cluster (takes 3-4 mins)..."
gcloud container clusters create griffin-dev \
    --zone=$ZONE \
    --num-nodes=2 \
    --machine-type=e2-standard-4 \
    --network=griffin-dev-vpc \
    --subnetwork=griffin-dev-wp

# ==========================================
# TASK 6: Prepare K8s Credentials & Secrets
# ==========================================
echo -e "\n[Task 6] Configuring Kubernetes secrets..."
gcloud container clusters get-credentials griffin-dev --zone=$ZONE

rm -rf wp-k8s
gsutil cp -r gs://spls/gsp511/wp-k8s .
cd wp-k8s

gcloud iam service-accounts keys create key.json \
    --iam-account=cloud-sql-proxy@$PROJECT_ID.iam.gserviceaccount.com

kubectl create secret generic cloudsql-instance-credentials \
    --from-file key.json

sed -i 's/username_to_be_replaced/wp_user/g' wp-env.yaml
sed -i 's/password_to_be_replaced/stormwind_rules/g' wp-env.yaml

kubectl apply -f wp-env.yaml

# ==========================================
# TASK 7: Deploy WordPress & Service
# ==========================================
echo -e "\n[Task 7] Deploying WordPress application..."
export MYSQL_CONN=$(gcloud sql instances describe griffin-dev-db --format='value(connectionName)')

sed -i "s/YOUR_SQL_INSTANCE/$MYSQL_CONN/g" wp-deployment.yaml

kubectl apply -f wp-deployment.yaml
kubectl apply -f wp-service.yaml

# ==========================================
# TASK 9: Grant IAM Access to User 2
# ==========================================
echo -e "\n[Task 9] Granting IAM Editor role to $USER2_EMAIL..."
if [ -n "$USER2_EMAIL" ]; then
    gcloud projects add-iam-policy-binding $PROJECT_ID \
        --member="user:$USER2_EMAIL" \
        --role="roles/editor"
else
    echo "⚠️ USER2_EMAIL was blank. Please run Task 9 IAM policy command manually."
fi

echo -e "\n=========================================================="
echo " Automations Complete! You can now click 'Check my progress'"
echo " for Tasks 1, 2, 3, 4, 5, 6, 7, and 9."
echo "=========================================================="
```

```bash
kubectl get svc wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```
