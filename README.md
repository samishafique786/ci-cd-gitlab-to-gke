# ci-cd-gitlab-to-gke

# CI/CD with Self-Hosted GitLab, CSC Cloud, and GKE

This repository will have a full CI/CD framework using a self-hosted GitLab instance deployed on the CSC - IT Center for Science cloud platform. It shows how automating the build, test, and deployment of a simple Node.js web application to Google Kubernetes Engine (GKE) works. I made this project from scratch and thought that It would be a learning experience for me to build a full workflow to showcase my DevOps, sys admin and Linux skills. 

## Project Goal

The primary goal was to establish a  CI/CD pipeline with the control and security benefits of a self-hosted GitLab server compared to SaaS providers like GitLab.com. The primary points are provisioning infrastructure, configuring GitLab, DNS/SSL, containerizing applications, and deployment to Kubernetes (GKE) automatically.

**Workflow:**

1.  **Push:** Developer pushes code changes to the `main` branch of the self-hosted GitLab repository.
2.  **Trigger:** GitLab CI/CD pipeline is triggered automatically.
3.  **Runner:** A GitLab Runner (installed on the GitLab server VM in this setup, or potentially on K8s) picks up the job.
4.  **Test:** The pipeline executes automated tests (e.g., unit/integration tests for the Node.js app).
5.  **Build:** A Docker image containing the application is built using the `app/Dockerfile`.
6.  **Push:** The built Docker image is pushed to the self-hosted GitLab Container Registry (secured with SSL).
7.  **Deploy:** The pipeline authenticates with GKE and uses `kubectl` to deploy (or update) the application using the newly pushed Docker image. A LoadBalancer service exposes the application.
8.  **Access:** External users access the deployed application via the GKE LoadBalancer's public IP address.

## Technologies Used

*   **Cloud Platform:** CSC - IT Center for Science (cPouta) for VM hosting/GKE for Kuberenetes (autopilot cluster)
*   **CI/CD & VCS:** GitLab Self-Hosted Community Edition
*   **Containerization:** Docker
*   **Orchestration:** Google Kubernetes Engine (GKE - Autopilot)
*   **Application:** Node.js, Express.js, SQLite (Simple Feedback App - **Code needs to be added by the user**)
*   **DNS:** DuckDNS (Free Dynamic DNS)
*   **SSL:** Let's Encrypt (via Certbot)
*   **Operating System:** Ubuntu 22.04 LTS
*   **IaC/Automation Tools:** `gcloud` CLI, `kubectl`

## Repository Structure

*   `.gitlab-ci.yml`: Defines the CI/CD pipeline stages and jobs.
*   `app/`: Contains the placeholder for the Node.js application code and its `Dockerfile`. **You need to add your actual application code here.**
*   `kubernetes/`: Contains example Kubernetes manifest files (`deployment.yaml`, `service.yaml`) for reference. The pipeline uses direct `kubectl` commands.
*   `docs/`: Contains supplementary documentation like the architecture diagram.
*   `README.md`: This file.

## Prerequisites

*   An account with CSC - IT Center for Science (or another cloud provider).
*   A Google Cloud Platform (GCP) account with billing enabled (GKE usage).
*   A DuckDNS account and chosen subdomain.
*   Domain name pointing to your GitLab server's public IP (or use DuckDNS).
*   Locally installed tools: `git`, `ssh`, `gcloud` CLI, `kubectl`, `certbot` (on the server).

## Setup Guide (High-Level Overview)

1.  **Provision VM (CSC cPouta):**
    *   Launch an Ubuntu 22.04 LTS VM (e.g., 4 vCPUs, 8GB RAM, 100GB storage).
    *   Configure Security Group: Allow inbound SSH (22), HTTP (80), HTTPS (443), GitLab Registry (5050).
    *   Assign a Floating IP.

2.  **Install GitLab Self-Hosted:**
    *   SSH into the VM.
    *   Update system: `sudo apt update && sudo apt upgrade -y`
    *   Install dependencies: `sudo apt install -y curl openssh-server ca-certificates postfix` (Postfix for email notifications).
    *   Add GitLab repository and install `gitlab-ce` (See thesis p. 27-29).

3.  **Configure DNS & SSL:**
    *   Point your DuckDNS subdomain (e.g., `your-gitlab.duckdns.org`) to the VM's Floating IP.
    *   Install Certbot: `sudo apt install certbot -y`.
    *   Stop GitLab temporarily: `sudo gitlab-ctl stop`.
    *   Obtain SSL certificate: `sudo certbot certonly --standalone -d your-gitlab.duckdns.org` (and potentially for `your-gitlab.duckdns.org:5050` if needed, though securing the registry path in `gitlab.rb` is typical).
    *   Edit GitLab configuration: `sudo nano /etc/gitlab/gitlab.rb`.
        *   Set `external_url 'https://your-gitlab.duckdns.org'`.
        *   Configure `nginx[...]` settings for SSL certificate paths (See thesis p. 32).
        *   Set `registry_external_url 'https://your-gitlab.duckdns.org:5050'`.
        *   Configure `registry_nginx[...]` settings for SSL (See thesis p. 33).
    *   Reconfigure and restart GitLab: `sudo gitlab-ctl reconfigure && sudo gitlab-ctl restart`.

4.  **GitLab Initial Setup:**
    *   Access `https://your-gitlab.duckdns.org` in your browser.
    *   Find the initial root password: `sudo cat /etc/gitlab/initial_root_password`.
    *   Login as `root` and change the password.
    *   (Optional) Configure GitLab settings (e.g., disable public sign-ups, enable user account creation if needed - see thesis p. 34).

5.  **Install GitLab Runner:**
    *   Install the runner on the same VM (for this project setup): Follow official GitLab Runner installation docs for Linux.
    *   Register the runner:
        *   Go to your GitLab project's `Settings > CI/CD > Runners`.
        *   Get the registration token and URL (`https://your-gitlab.duckdns.org`).
        *   Run `sudo gitlab-runner register` and follow prompts (use `docker` executor if pipeline uses Docker).

6.  **Setup GCP & GKE:**
    *   Install `gcloud` CLI locally.
    *   Authenticate: `gcloud auth login`.
    *   Create a GCP project or use an existing one. Set it: `gcloud config set project YOUR_PROJECT_ID`.
    *   Enable Kubernetes Engine API in GCP Console.
    *   Create GKE Autopilot cluster (e.g., via GCP Console or `gcloud container clusters create-auto...` - see thesis p. 41).
    *   Install `kubectl`: `gcloud components install kubectl`.
    *   Install GKE auth plugin: `sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin`.
    *   Get cluster credentials: `gcloud container clusters get-credentials CLUSTER_NAME --zone CLUSTER_ZONE`.
    *   Verify connection: `kubectl get nodes`.

7.  **Configure Kubernetes for GitLab Registry:**
    *   Create a GitLab Deploy Token or Personal Access Token (`read_registry` scope).
    *   Create a Kubernetes secret to pull images from the private GitLab registry:
        ```bash
        kubectl create secret docker-registry gitlab-registry-secret \
          --docker-server=your-gitlab.duckdns.org:5050 \
          --docker-username=<your_gitlab_username_or_deploy_token_username> \
          --docker-password=<your_gitlab_token> \
          --docker-email=<your_email> # Optional
        ```
    *   Patch the default service account to use this secret (or create a dedicated one):
        ```bash
        kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "gitlab-registry-secret"}]}'
        ```

8.  **Prepare Application Code:**
    *   Clone this repository.
    *   **Crucially: Replace the contents of the `app/` directory with your actual Node.js application code.** Ensure it includes a `package.json` and your main server file (e.g., `server.js`).
    *   Customize the `app/Dockerfile` if necessary for your application.

9.  **Configure CI/CD Variables in GitLab:**
    *   Go to your GitLab project's `Settings > CI/CD > Variables`.
    *   Add the following variables (as defined in `.gitlab-ci.yml` and thesis p. 45-46):
        *   `CI_REGISTRY`: `your-gitlab.duckdns.org:5050` (Verify this - the pipeline file has it hardcoded, but good practice to have it as a variable)
        *   `DEPLOYMENT_NAME`: `my-app` (or your desired K8s deployment name)
        *   `SERVICE_NAME`: `my-app-service` (or your desired K8s service name)
        *   `CONTAINER_NAME`: `imtiaz-node-app` (container name within the pod)
        *   `APP_PORT`: `3000` (port your app listens on inside the container)
        *   `CI_REGISTRY_USER`: `<gitlab_username_or_deploy_token_username>`
        *   `CI_REGISTRY_PASSWORD`: (Masked) `<gitlab_token>`
        *   `GCP_SERVICE_ACCOUNT_KEY`: (Masked, File type recommended) Base64 encoded content of your GCP service account key JSON file (`cat key.json | base64 -w 0`). Ensure this service account has roles like `Kubernetes Engine Developer` (`roles/container.developer`).
        *   `GKE_PROJECT_ID`: `YOUR_GCP_PROJECT_ID`
        *   `GKE_CLUSTER_NAME`: `YOUR_GKE_CLUSTER_NAME`
        *   `GKE_REGION`: `YOUR_GKE_CLUSTER_REGION` (e.g., `us-central1`)

10. **Push and Run:**
    *   Commit your application code and the repository files (`.gitlab-ci.yml`, etc.).
    *   Push to the `main` branch of your self-hosted GitLab repository.
    *   Monitor the pipeline execution in the GitLab CI/CD section.
