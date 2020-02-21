# Tabs VS Spaces - GKE + Cloud SQL

This is based on this codelab <https://gabi.fyi/postgresql-gke>

## Project Configuration

Enable the following APIs, either via GUI or via `cloudshell`:

* Enable Cloud Shell API
* Enable Container Registry API
* Enable Kubernetes Engine API
* Enable Cloud Build API
* Enable Cloud SQL Admin API
* Enable Cloud Run API

```sh
gcloud services enable cloudshell.googleapis.com \
  cloudbuild.googleapis.com \
  sqladmin.googleapis.com \
  containerregistry.googleapis.com \ 
  container.googleapis.com \
  run.googleapis.com --async
```

First on your `.bashrc` on cloudshell create the environment variable `$MY_DB_PASSWORD`:

```sh
echo "export MY_DB_PASSWORD=thisIsAStrongPassword" >> ~/.bashrc
```

Reopen `cloudshell` to make sure the profile was loaded with the new ENV.

## Cloud SQL

Provision Cloud SQL Instance with HA

```sh
gcloud sql instances create --zone us-central1-f --database-version MYSQL_5_7 --tier db-n1-standard-4 --enable-bin-log --availability-type REGIONAL quizzes
```

Create `quizzes` schema

```sh
gcloud sql databases create results --instance=quizzes
```

Set `root` password

```sh
gcloud sql users set-password root --instance quizzes --host '127.0.0.1' --password $MY_DB_PASSWORD
```

## IAM & Authentication

Set up proxy credentials (Cloud SQL Client permission) as `proxy-user`

```sh
gcloud iam service-accounts create proxy-user --display-name "proxy-user"
```

Give permissions to service account

```sh
gcloud projects add-iam-policy-binding $(gcloud config get-value core/project) --member serviceAccount:proxy-user@$(gcloud config get-value core/project).iam.gserviceaccount.com --role roles/cloudsql.client
```

Generate `key.json`

```sh
gcloud iam service-accounts keys create key.json --iam-account proxy-user@$(gcloud config get-value core/project).iam.gserviceaccount.com
```

## Application

Download `cloud_sql_proxy` and make it runnable

```sh
wget https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -O cloud_sql_proxy && chmod +x cloud_sql_proxy
```

Connect the `cloud_sql_proxy`

```sh
./cloud_sql_proxy -instances=$(gcloud config get-value core/project):us-central1:quizzes=tcp:3306 -credential_file=key.json &
```

Download **TabsVsSpaces** from github and enter the folder

```sh
git clone --single-branch --branch cloudsql-sample-mysql https://github.com/gabidavila/php-docs-samples.git && cp -R php-docs-samples/cloud_sql/tabs-vs-spaces tabs-vs-spaces && cd tabs-vs-spaces
```

Run application locally

```sh
DB_USER=root DB_PASS=$MY_DB_PASSWORD DB_NAME=results php -S 0.0.0.0:8080 -d error_reporting=0
```

## Containerize

Build the docker image

```sh
docker build . -t quizzes
```

Run the docker image and show TabsVsSpaces app working

```sh
docker run --net="host" -d --rm --name runtime -e "DB_USER=root" -e "DB_PASS=$MY_DB_PASSWORD" -e "DB_NAME=results" quizzes
```

Kill `cloud_sql_proxy` and stop docker

```sh
killall cloud_sql_proxy && docker stop runtime
```

_Push_ to **Google Container Registry** using Google Cloud Build

```sh
gcloud builds submit --tag gcr.io/$(gcloud config get-value core/project)/quizzes
```

## Deploy

For this part we can deploy either using Google Kubernetes Engine or Google Cloud Run. Choose what path you want to follow.

### Deploy on GKE

Create the cluster

```sh
gcloud container clusters create php-cluster --zone us-central1-f  --machine-type=e2-highcpu-2  --enable-autorepair --enable-autoscaling --max-nodes=10 --min-nodes=1
```

List Clusters

```sh
gcloud container clusters list
```

Enable `kubectl` to use the cluster created on the previous step

```sh
gcloud container clusters get-credentials php-cluster --zone us-central1-f
```

Create secrets for kubernetes

```sh
kubectl create secret generic csql-proxy-acct --from-file=credentials.json=../key.json && kubectl create secret generic csql-secrets --from-literal=username=root  --from-literal=password=$MY_DB_PASSWORD --from-literal=dbname=results
```

---

**IMPORTANT**

**
**

**Edit `quizzes_deployment.yml` file replacing `PROJECT_ID` and `INSTANCE_CONNECTION_NAME`**

---

Create the deployment ("Workload")

```sh
kubectl create -f quizzes_deployment.yml
```

Enable horizontal pod autoscaling

```sh
kubectl autoscale deployment quizzes --min=3 --max=10 --cpu-percent=50
```

To see the pods being created execute

```sh
kubectl get pods
```

Expose the deployment

```sh
kubectl expose deployment quizzes --type "LoadBalancer" --port 80 --target-port 8080
```

Use `kubectl` to check the status of the `LoadBalancer`

```sh
kubectl describe services quizzes
```