# Implement-Cloud-Security-Fundamentals-on-Google-Cloud

🧩 Task 1 – Create a custom security role
#role-definition.yaml
 title: "orca_storage_editor_871"
 description: "Edit access for App Versions"
 stage: "ALPHA"
 includedPermissions:
  - storage.objects.get
  - storage.objects.list
  - storage.objects.create
  - storage.objects.update
  - storage.buckets.get

#Create the custom role
 export PROJECT_ID=$(gcloud config get-value project)
 gcloud iam roles create orca_storage_editor_871 \
  --project "$PROJECT_ID" \
  --file role-definition.yaml
  #✅ This matches the expected output format from the lab (only PROJECT_ID will differ).

👤 Task 2 & 3 – Create service account and bind roles
Create the service account
 export PROJECT_ID=$(gcloud config get-value project)
export SA_NAME="orca-private-cluster-724-sa"
export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create "$SA_NAME" \
  --display-name "GKE private cluster service account"


Bind required IAM roles
# Replace orca_storage_editor_871 if you change the custom role name.
  # 1. Logging – log writer
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/logging.logWriter"

# 2. Monitoring – metric writer
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/monitoring.metricWriter"

# 3. Monitoring – viewer (lab checker is strict about this one)
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "roles/monitoring.viewer"

# 4. Custom storage role
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member "serviceAccount:${SA_EMAIL}" \
  --role "projects/${PROJECT_ID}/roles/orca_storage_editor_871"


☸️ Task 4 – Create and configure a new GKE private cluster
 Set region / zone (for consistency)
 gcloud config set compute/region us-central1
 gcloud config set compute/zone us-central1-c

 export REGION=us-central1
 export ZONE=us-central1-c
 export PROJECT_ID=$(gcloud config get-value project)
 export SA_NAME="orca-private-cluster-724-sa"
 export SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

Create the private cluster (with adjusted disk type / size)
  This version avoids SSD quota issues by using pd-standard and --num-nodes=1.
 Adjust node count as needed in real environments.
 gcloud container clusters create orca-cluster-868 \
  --region "$REGION" \
  --network orca-build-vpc \
  --subnetwork orca-build-subnet \
  --service-account "$SA_EMAIL" \
  --enable-ip-alias \
  --enable-private-nodes \
  --enable-private-endpoint \
  --enable-master-authorized-networks \
  --disk-type=pd-standard \
  --num-nodes=1


Add jumphost internal IP to master authorized network
From Cloud Shell (or any host with gcloud and access):

# Get jumphost IP (if you don’t want to hard-code it)
JUMPHOST_IP=$(gcloud compute instances describe orca-jumphost \
  --zone us-central1-c \
  --format='get(networkInterfaces[0].networkIP)')

# Or hard-code if you know it, e.g.
# JUMPHOST_IP=192.168.10.2

gcloud container clusters update orca-cluster-868 \
  --region "$REGION" \
  --enable-master-authorized-networks \
  --master-authorized-networks "${JUMPHOST_IP}/32"

🧱 Task 5 – Deploy an app to the private cluster (via jumphost
 On Cloud Shell – SSH into jumphost
  gcloud compute ssh orca-jumphost --zone us-central1-c

All commands below run on the jumphost.
 1. Install and enable gke-gcloud-auth-plugin
  sudo apt-get update
 sudo apt-get install -y google-cloud-sdk-gke-gcloud-auth-plugin

 echo "export USE_GKE_GCLOUD_AUTH_PLUGIN=True" >> ~/.bashrc
 source ~/.bashrc

  2. Get cluster credentials using internal IP
   export PROJECT_ID=$(gcloud config get-value project)

gcloud container clusters get-credentials orca-cluster-868 \
  --internal-ip \
  --region us-central1 \
  --project "$PROJECT_ID"

Quick check:
 kubectl get nodes

3. Deploy the test application

 Optional: expose via LoadBalancer for testing:

 kubectl expose deployment hello-server \
  --type=LoadBalancer \
  --port 80 \
  --target-port 8080

kubectl get service hello-server
