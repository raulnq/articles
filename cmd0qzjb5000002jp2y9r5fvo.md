---
title: "How to Host n8n for Free on Google Cloud Platform"
datePublished: Sat Jul 12 2025 21:18:28 GMT+0000 (Coordinated Universal Time)
cuid: cmd0qzjb5000002jp2y9r5fvo
slug: how-to-host-n8n-for-free-on-google-cloud-platform
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1752279044615/e401e879-f0b0-45cc-a94c-94392ee7f09e.png
tags: gcp, n8n

---

[n8n](https://n8n.io/) is a powerful workflow automation tool that has gained significant popularity in recent years. It's a free and open-source alternative to tools like [Zapier](https://zapier.com/) or [Microsoft Power Automate](https://www.microsoft.com/en-us/power-platform/products/power-automate). It lets us connect different services and automate repetitive tasks using a visual, node-based interface.

In this post, we will learn how to deploy n8n on a [Google Cloud Platform](https://cloud.google.com/?hl=en) (GCP) VM using [Infrastructure Manager](https://blog.raulnq.com/terraform-as-a-service-with-google-clouds-infrastructure-manager?source=more_articles_bottom_blogs) (Terraform-as-a-Service), [Secret Manager](https://cloud.google.com/secret-manager/docs/overview), and [Cloudflare Tunnel](https://blog.raulnq.com/exposing-gcp-vms-to-the-internet-with-cloudflare-tunnel) to expose it securely to the internet, with zero public IP required.

## Prerequisites

Before we begin, we will need:

* A [Google Cloud Project](https://developers.google.com/workspace/guides/create-project)**.**
    
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) installed and authenticated.
    
* A [Cloudflare Account](https://www.cloudflare.com/) with a registered domain.
    
* A [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/).
    
* A [Service Account](https://cloud.google.com/iam/docs/service-accounts-create) with Admin Role (`roles/admin`).
    

## **APIs**

We need to enable the Infrastructure Manager API, the Compute Engine API, the Cloud Storage API, and the Secret Manager API:

```powershell
gcloud services enable config.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable secretmanager.googleapis.com
```

## Terraform Script

Create a `terraform/main.tf` file with the following content:

```json
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = ">= 4.34.0"
    }
  }
}

variable "project_id" {
  type        = string
  description = "The GCP project ID to deploy resources into."
}

variable "tunnel_id" {
  type        = string
  description = "The Cloudflare tunnel ID."
}

variable "cloudflare_token" {
  type        = string
  description = "The Cloudflare token."
  sensitive   = true
}

variable "domain" {
  type        = string
  description = "The domain to expose n8n on."
}

resource "google_secret_manager_secret" "cloudflare_token" {
  project   = var.project_id
  secret_id = "cloudflare-token"

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "cloudflare_token_version" {
  secret      = google_secret_manager_secret.cloudflare_token.id
  secret_data = var.cloudflare_token
}

resource "google_project_iam_member" "secret_accessor" {
  project = var.project_id
  role    = "roles/secretmanager.secretAccessor"
  member  = "serviceAccount:${google_compute_instance.free_tier_vm.service_account[0].email}"
}

resource "google_compute_instance" "free_tier_vm" {
  project      = var.project_id
  machine_type = "e2-micro"
  zone         = "us-central1-a"
  name         = "free-vm"
  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }
  network_interface {
    network = "default"
    access_config {}
  }
  metadata = {
    startup-script = templatefile("${path.module}/startup-script.sh", {
      secret_id = google_secret_manager_secret.cloudflare_token.secret_id
      domain    = var.domain
      tunnel_id = var.tunnel_id
    })
  }
  service_account {
    scopes = ["cloud-platform"]
  }
}
```

The Terraform script defines the infrastructure required to deploy n8n on a GCP VM, securely store sensitive data, and configure access permissions. Let's break down our configuration:

* [`google_secret_manager_secret`](https://registry.terraform.io/providers/hashicorp/google/4.75.1/docs/resources/secret_manager_secret): Creates a secret in Secret Manager for storing our Cloudflare token. The `auto` replication ensures the secret is automatically replicated across regions for high availability.
    
* [`google_secret_manager_secret_version`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/data-sources/secret_manager_secret_version): Adds the actual token content as a versioned secret.
    
* [`google_project_iam_member`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam#google_project_iam_member): This grants our VM's service account the necessary permissions to access the secret we created.
    
* [`google_compute_instance`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance): This creates our VM with the following specifications:
    
    * **Machine Type**: `e2-micro`.
        
    * **Zone**: `us-central1-a`.
        
    * **Operating System**: Debian 11
        
    * **Network**: Uses the default VPC with external IP.
        
    * **Startup Script**: Templated script that receives our configuration variables.
        
    * **Service Account**: Configured with `cloud-platform` scope for necessary permissions.
        

## Startup Script

Create the `terraform/startup-script.sh` file and add the following code snippets.

```bash
apt-get update -y
apt-get install -y curl lsb-release
```

This ensures our system is up-to-date and has the basic tools needed for the installation.

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared.deb
rm cloudflared.deb
```

Downloads and installs the latest version of `cloudflared`, which creates the secure tunnel to Cloudflare.

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs
npm install -g n8n
```

Installs Node.js `22.x` and then n8n globally via npm.

```bash
mkdir -p /opt/n8n
useradd -r -d /opt/n8n -s /bin/false n8n
chown -R n8n:n8n /opt/n8n
```

Creates a directory `/opt/n8n` for n8n data. It also creates a system user `n8n` with no login shell for security and sets the ownership of `/opt/n8n` to this new user.

```bash
cat > /etc/systemd/system/n8n.service <<EOF
[Unit]
Description=n8n
Requires=network.target
After=network.target

[Service]
Type=simple
User=n8n
Group=n8n
WorkingDirectory=/opt/n8n
ExecStart=/usr/bin/n8n start
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

Defines a `systemd` service file to run n8n, ensuring it starts after the network and restarts on failure.

```bash
mkdir -p /etc/cloudflared/
cat > /etc/cloudflared/config.yml << EOF
tunnel: ${tunnel_id}
credentials-file: /root/.cloudflared/${tunnel_id}.json
ingress:
  - hostname: ${domain}
    service: http://127.0.0.1:5678
  - service: http_status:404
EOF
```

Creates a configuration file specifying the tunnel ID, credentials file location, and ingress rules to route traffic from our domain to n8n´s default port (`5678`), with a fallback to a `404` status for unmatched requests.

```bash
CLOUDFLARE_TOKEN=$(gcloud secrets versions access latest --secret="${secret_id}")
cloudflared service install $CLOUDFLARE_TOKEN --config /etc/cloudflared/config.yml
```

Retrieves the Cloudflare token from Secret Manager and installs the tunnel as a service.

```bash
systemctl start cloudflared
systemctl enable cloudflared
systemctl start n8n
systemctl enable n8n
```

Ensures both n8n and the tunnel start automatically on boot.

## Deployment Steps

Create the `terraform/inputs.tfvars` file to store all our parameter values:

```powershell
project_id       = "<MY_PROJECT_ID>"
tunnel_id        = "<MY_TUNNEL_ID>"
domain           = "<MY_DOMAIN>"
cloudflare_token = "<MY_CLOUDFLARE_TOKEN>"
```

Run the following command to start the deployment:

```powershell
gcloud infra-manager deployments apply n8n-deployment --location=us-central1 --local-source=./terraform --service-account=projects/<MY_PROJECT_ID> /serviceAccounts/<MY_SERVICE_ACCOUNY>@<MY_PROJECT_ID>x.iam.gserviceaccount.com --inputs-file=./terraform/inputs.tfvars
```

The deployment will finish in less than 1 minute, but the whole startup script might take around 45 minutes to complete due to the limited resources of our VM. We can track the progress by checking the VM logs:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1752335710553/832d7d5c-5ac2-4f61-af9c-71c17aff5c7c.png align="center")

Once it is completed, we can go to our domain and start setting up n8n:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1752335793173/5839f7bd-c6ce-4392-86ad-2c0ec10a9e73.png align="center")

If you're new to workflow automation or want complete control over your automation platform, this setup is an excellent starting point with minimal cost. You can find all the code [here](https://github.com/raulnq/gcp-infrastructure-manage/tree/n8n/terraform). Thanks, and happy coding.