# GKE Launch and Manual Deployment Runbook

## Project: `august2027` \| Cluster: `rushi-gke` \| Application: Flipkart

This runbook explains how to stop the Jenkins VM temporarily, recreate
the GKE cluster with the same name, and deploy the latest application
image from Docker Hub without depending on Jenkins.

> **Important:** Stopping Jenkins does not build or publish a new Docker
> image. This runbook deploys the latest image that has already been
> pushed to Docker Hub. If source code changes need a new image, build
> and push it from another machine before deploying.

------------------------------------------------------------------------

## 1. Current project details

  ----------------------------------------------------------------------------------------------------------------
  Item                                Value
  ----------------------------------- ----------------------------------------------------------------------------
  GCP project ID                      `august2027`

  GKE cluster name                    `rushi-gke`

  GKE location                        Zone `us-central1-f`

  Node count                          2

  Kubernetes deployment               `flipkart-app`

  Kubernetes service                  `flipkart-service`

  Docker Hub image                    `devopsbyrushi/octsecurebankcicdpipleline:v1`

  GitHub repository                   `https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git`

  WAR file                            `target/flipkart-clone.war`

  Kubernetes manifests folder         `k8s/`
  ----------------------------------------------------------------------------------------------------------------

The commands below retain these names to make the lab easy to follow.

## 2. Understand the workflow

``` text
GitHub source code
       |
       | (build and push image when code changes)
       v
Docker Hub
devopsbyrushi/octsecurebankcicdpipleline:v1
       |
       | Cloud Shell / kubectl pulls image
       v
GKE cluster: rushi-gke
       |
       v
Deployment: flipkart-app
       |
       v
Application Pods
       |
       v
Service: flipkart-service
       |
       v
Application external IP
```

Jenkins is not needed for the GKE creation or deployment steps below.
Cloud Shell runs the `gcloud` and `kubectl` commands.

------------------------------------------------------------------------

## 3. Before stopping Jenkins

### 3.1 Confirm the Docker image exists

Open Docker Hub and confirm that this repository and tag exist:

`devopsbyrushi/octsecurebankcicdpipleline:v1`

If the image must contain newer application code, build and push it
**before** stopping Jenkins, or use another computer with Docker
installed.

Example commands from a machine that has the Git repository and Docker
installed:

``` bash
git clone https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git
cd rushiDevOpsTraining-FlipkartWarDemo

mvn clean install
docker build -t devopsbyrushi/octsecurebankcicdpipleline:v1 .
docker login
docker push devopsbyrushi/octsecurebankcicdpipleline:v1
```

Enter your Docker Hub credentials when `docker login` asks. Do not put
the password directly in a command.

### 3.2 Record the Jenkins VM details

In Google Cloud Console:

1.  Open **Compute Engine → VM instances**.
2.  Find `jenkins-server`.
3.  Record its current external IP and internal IP.
4.  Record the VM zone and machine settings if you may need to resize or
    recreate it later.

### 3.3 Stop the Jenkins VM

In **Compute Engine → VM instances**:

1.  Select `jenkins-server`.
2.  Click **Stop**.
3.  Confirm the operation.

Stopping the VM stops Jenkins and any services hosted on that VM,
including the SonarQube, Prometheus and Grafana containers if they are
running there. Persistent disks normally remain attached and retain
their data.

**External IP warning:** If the VM uses an ephemeral external IP, its
external IP can change after it is stopped and started. A reserved
static external IP avoids this change, but may incur charges while
unused depending on current Google Cloud pricing. Check the IP type in
the VM's network interface before stopping.

------------------------------------------------------------------------

## 4. Open Google Cloud Shell

1.  Open [Google Cloud Console](https://console.cloud.google.com/).
2.  Select project **`august2027`**.
3.  Click the **Activate Cloud Shell** icon in the top-right.
4.  Wait for the terminal to open.

All commands in Sections 5--10 are run in **Google Cloud Shell**, unless
another location is explicitly mentioned.

## 5. Set and verify the GCP project

``` bash
gcloud config set project august2027
```

Verify:

``` bash
gcloud config get-value project
```

Expected:

``` text
august2027
```

Enable the Kubernetes Engine API if it is not already enabled:

``` bash
gcloud services enable container.googleapis.com
```

## 6. Create the GKE cluster with the same name

The previous lab cluster used the name `rushi-gke`, zone
`us-central1-f`, and two nodes.

Run:

``` bash
gcloud container clusters create rushi-gke \
  --zone=us-central1-f \
  --num-nodes=2 \
  --project=august2027
```

Wait for the command to finish. Cluster creation can take several
minutes.

> If Google Cloud reports that a cluster named `rushi-gke` already
> exists, do not run the create command again. Check it with
> `gcloud container clusters list` and use the existing cluster.

### 6.1 Confirm the cluster

``` bash
gcloud container clusters list \
  --project=august2027
```

## 7. Connect Cloud Shell to the cluster

Get cluster credentials:

``` bash
gcloud container clusters get-credentials rushi-gke \
  --zone=us-central1-f \
  --project=august2027
```

Check the nodes:

``` bash
kubectl get nodes
```

Both nodes should eventually show `Ready`.

------------------------------------------------------------------------

## 8. Get the Kubernetes YAML files

The application manifests are stored in the `k8s/` folder of the GitHub
repository.

Clone the repository in Cloud Shell:

``` bash
git clone https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git
```

Move into the repository:

``` bash
cd rushiDevOpsTraining-FlipkartWarDemo
```

Check the files:

``` bash
ls
ls k8s/
```

You should see the Kubernetes YAML files inside `k8s/`.

If the repository was already cloned in this Cloud Shell session, update
it instead:

``` bash
cd ~/rushiDevOpsTraining-FlipkartWarDemo
git pull
```

If the folder is not present, run the `git clone` command above.

## 9. Check the image and deployment configuration

Check which image is configured in the deployment YAML:

``` bash
grep -n "image:" k8s/deployment.yaml
```

The deployment should use:

``` text
devopsbyrushi/octsecurebankcicdpipleline:v1
```

Also check that the container uses `imagePullPolicy: Always`. This is
important when reusing the same `v1` tag:

``` yaml
image: devopsbyrushi/octsecurebankcicdpipleline:v1
imagePullPolicy: Always
```

If `imagePullPolicy` is missing or set to `IfNotPresent`, edit
`k8s/deployment.yaml` and set it to `Always` under the application
container. Save the file.

> If your manifest has a different filename, use the actual deployment
> YAML filename shown by `ls k8s/`.

## 10. Deploy the application

Apply the Kubernetes YAML files:

``` bash
kubectl apply -f k8s/
```

Restart the deployment so the pods pull the image again:

``` bash
kubectl rollout restart deployment/flipkart-app
```

Wait for the rollout:

``` bash
kubectl rollout status deployment/flipkart-app --timeout=300s
```

If successful, Kubernetes reports that the deployment has successfully
rolled out.

### 10.1 Verify pods and deployment

``` bash
kubectl get deployments
kubectl get pods -o wide
```

All application pods should show `Running` and become `Ready`.

If a pod is not starting, inspect it:

``` bash
kubectl describe pod POD_NAME
```

Replace `POD_NAME` with the actual pod name from `kubectl get pods`.

View application logs:

``` bash
kubectl logs deployment/flipkart-app
```

## 11. Get the application external IP

Check the service:

``` bash
kubectl get services
```

Find `flipkart-service`. Wait until its `EXTERNAL-IP` changes from
`<pending>` to an IP address.

You can also watch the service:

``` bash
kubectl get service flipkart-service --watch
```

Press `Ctrl+C` after the external IP appears.

Open the application in a browser:

``` text
http://EXTERNAL_IP/
```

Replace `EXTERNAL_IP` with the address displayed by
`kubectl get services`.

If the service has no external IP, inspect it:

``` bash
kubectl describe service flipkart-service
```

------------------------------------------------------------------------

## 12. How to deploy a newer Docker image later

When application code changes, a new image must first be built and
pushed to Docker Hub. Jenkins is not running during this manual process,
so use a separate machine with Git, Maven and Docker installed.

### Option A --- Keep using the existing `v1` tag

On the build machine, build and push:

``` bash
mvn clean install
docker build -t devopsbyrushi/octsecurebankcicdpipleline:v1 .
docker login
docker push devopsbyrushi/octsecurebankcicdpipleline:v1
```

Then, in Cloud Shell, connect to GKE if necessary and run:

``` bash
kubectl rollout restart deployment/flipkart-app
kubectl rollout status deployment/flipkart-app --timeout=300s
```

This works when the Deployment has `imagePullPolicy: Always`.

### Option B --- Use a new version tag (recommended)

For example, build and push `v2` from the build machine:

``` bash
docker build -t devopsbyrushi/octsecurebankcicdpipleline:v2 .
docker push devopsbyrushi/octsecurebankcicdpipleline:v2
```

Then, in Cloud Shell:

``` bash
kubectl set image deployment/flipkart-app \
  CONTAINER_NAME=docker.io/devopsbyrushi/octsecurebankcicdpipleline:v2
```

Replace `CONTAINER_NAME` with the actual container name from:

``` bash
kubectl get deployment flipkart-app \
  -o jsonpath='{.spec.template.spec.containers[*].name}'
```

Wait for the update:

``` bash
kubectl rollout status deployment/flipkart-app --timeout=300s
```

Using a unique tag for each release makes it easier to identify and roll
back application versions.

------------------------------------------------------------------------

## 13. Jenkins VM IP address --- where it must be updated

The Jenkins VM IP is **not required** to create the GKE cluster or
deploy the application manually from Cloud Shell. Update it only in
services or configurations that communicate with Jenkins.

### 13.1 Jenkins URL and browser bookmarks

If the Jenkins VM receives a new external IP after restart, access
Jenkins using:

``` text
http://NEW_JENKINS_EXTERNAL_IP:8080
```

Update any saved bookmarks or external integrations that use the old IP.

### 13.2 SonarQube webhook

If SonarQube is running on the Jenkins VM, it is also stopped when the
VM is stopped.

When Jenkins and SonarQube are running again:

1.  Open SonarQube.
2.  Go to **Administration → Configuration → Webhooks**.
3.  Edit the Jenkins webhook.
4.  Set its URL to:

``` text
http://NEW_JENKINS_ADDRESS:8080/sonarqube-webhook/
```

Use an address that SonarQube can reach. If both services are on the
same VM, `http://localhost:8080/sonarqube-webhook/` may be suitable. If
they are separate, use the reachable private or public IP and correct
port.

### 13.3 Prometheus Jenkins target

If Prometheus is configured to scrape Jenkins using the Jenkins VM's
private IP, check:

``` text
/opt/monitoring/prometheus/prometheus.yml
```

Look for a target similar to:

``` yaml
static_configs:
  - targets: ['OLD_JENKINS_PRIVATE_IP:8080']
```

Replace it with the current Jenkins **private IP** if that IP has
changed.

Then reload or recreate Prometheus using the monitoring Compose
directory:

``` bash
cd /opt/monitoring
docker compose up -d --force-recreate prometheus
```

This step is done on the Jenkins/monitoring VM after it is running
again, not in Cloud Shell.

### 13.4 Jenkins configuration and other integrations

Search for the old Jenkins IP in any external scripts, monitoring
configurations, webhook destinations, or integrations that call Jenkins
directly. Update only entries that actually point to the old IP.

**Important distinction:** - Jenkins external IP: used to access Jenkins
from outside. - Jenkins private IP: used by services communicating
inside the GCP network, such as Prometheus on the same VM. - GKE
application external IP: belongs to the Kubernetes LoadBalancer service
and is separate from the Jenkins VM IP.

------------------------------------------------------------------------

## 14. Quick command checklist

Run these commands in Cloud Shell when you want to recreate the cluster
and deploy the application:

``` bash
# 1. Select project
gcloud config set project august2027

# 2. Enable GKE API
gcloud services enable container.googleapis.com

# 3. Create cluster
gcloud container clusters create rushi-gke \
  --zone=us-central1-f \
  --num-nodes=2 \
  --project=august2027

# 4. Connect to cluster
gcloud container clusters get-credentials rushi-gke \
  --zone=us-central1-f \
  --project=august2027

# 5. Clone application repository
git clone https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git

# 6. Enter repository
cd rushiDevOpsTraining-FlipkartWarDemo

# 7. Deploy Kubernetes manifests
kubectl apply -f k8s/

# 8. Restart application to pull latest image
kubectl rollout restart deployment/flipkart-app

# 9. Wait for deployment
kubectl rollout status deployment/flipkart-app --timeout=300s

# 10. Verify resources
kubectl get nodes
kubectl get deployments
kubectl get pods -o wide
kubectl get services
```

If the repository already exists in Cloud Shell, use `git pull` in its
folder instead of cloning it again.

------------------------------------------------------------------------

## 15. Important notes

1.  Deleting a GKE cluster and stopping a Jenkins VM are separate
    operations.
2.  Stopping the Jenkins VM does not delete its persistent disk, but it
    also stops SonarQube, Prometheus and Grafana if they are hosted on
    that VM.
3.  A stopped VM may receive a different ephemeral external IP when
    started again. Consider reserving a static IP if a stable address is
    required.
4.  Recreating the GKE cluster creates new Kubernetes nodes and a new
    LoadBalancer IP. The application URL may therefore change.
5.  Kubernetes deployments and services are recreated by applying the
    repository's `k8s/` manifests.
6.  This runbook assumes the Docker Hub repository is public. If it is
    private, create a Kubernetes image pull secret and reference it in
    the Deployment.
7.  The commands assume the cluster is in `us-central1-f` and has two
    nodes. If the original cluster used custom networking, machine
    types, release channels, or other settings, add those same settings
    when recreating it.
8.  Do not delete persistent disks, static IPs, or other resources
    unless you have confirmed they are no longer needed.

**End result:** Jenkins can remain stopped while you create `rushi-gke`,
deploy the Docker Hub image, verify the pods, and access the application
through the GKE LoadBalancer external IP.
