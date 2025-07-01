---
title: "Exposing GCP VMs to the Internet with Cloudflare Tunnel"
datePublished: Tue Jul 01 2025 21:48:49 GMT+0000 (Coordinated Universal Time)
cuid: cmcl287ba000102i57bnih5wp
slug: exposing-gcp-vms-to-the-internet-with-cloudflare-tunnel
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1751383415511/47f4018f-2672-48a6-ab3e-97dc363567f6.png
tags: cloudflare, google-cloud, gcp, cloudflare-tunnel

---

In the previous [article](https://blog.raulnq.com/terraform-as-a-service-with-google-clouds-infrastructure-manager), we deployed a virtual machine (VM) on Google Cloud Platform (GCP) with a simple NGINX web server. While functional, exposing a server directly to the internet with a public IP address and firewall rules can introduce security vulnerabilities and management overhead. Today, we will build on our previous setup and introduce a more secure and robust way to expose our web server: [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).

Cloudflare Tunnel creates a secure, outbound-only connection between our VM and the Cloudflare network. A lightweight daemon, `cloudflared`, runs on our VM and establishes this connection. This means we can have a VM running in our GCP project without a public IP address, making it inaccessible from the public internet and less vulnerable to direct attacks.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1751388485920/095fadd5-05fd-4b07-b755-3a0fe965368e.png align="center")

Here are some key benefits of using Cloudflare Tunnel:

* **Enhanced security:** Our server's IP address is hidden, which protects it from direct attacks.
    
* **No inbound ports:** We no longer need to open ports on our firewall.
    
* **DDoS protection:** All traffic to our server goes through Cloudflare's network, which offers DDoS protection.
    
* **Free tier:** Cloudflare provides a generous free tier for its Tunnel service, making it a cost-effective solution for personal projects and small applications.
    

## Pre-requisites

* A [Google Cloud Project](https://developers.google.com/workspace/guides/create-project)**.**
    
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) installed and authenticated.
    
* A [Cloudflare Account](https://www.cloudflare.com/) with a registered domain.
    
* Download the initial [code](https://github.com/raulnq/gcp-infrastructure-manage).
    

## Updating the Terraform Configuration

First, let's update our Terraform code to remove the public IP address and the firewall rule since they are no longer needed. Modify the `terraform/main.tf` file as shown below:

```json
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = ">= 4.34.0"
    }
  }
}

resource "google_compute_instance" "free_tier_vm" {
  project      = var.project_id
  machine_type = "e2-micro"
  zone         = "us-central1-a"
  name         = "my-first-vm"
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
    startup-script = file("${path.module}/startup-script.sh")
  }
}

variable "project_id" {
  type        = string
  description = "The GCP project ID to deploy resources into."
}
```

Notice that the `tags` property has been removed from the `google_compute_instance` resource, and the entire `google_compute_firewall` resource has been deleted. We need to keep the `access_config` block to assign a public IP address to the VM so it can still access the internet.

## Setting up Cloudflare Tunnel

Setting up a Cloudflare Tunnel involves a few manual steps that we only need to do once.

1. Install [`cloudflared`](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/) on our local machine.
    
2. Authenticate with Cloudflare by running `cloudflared tunnel login`.
    
3. Create a tunnel by running `cloudflared tunnel create my-gcp-tunnel`.
    
4. Create a `CNAME` record in Cloudflare DNS to link our subdomain to our tunnel:
    

```powershell
cloudflared tunnel route dns my-gcp-tunnel nginx.<MY_DOMAIN>
```

## Updating the Startup Script

Now, let's update the `terraform/startup-script.sh` to install and run `cloudflared` on the VM when it starts. To get the token and ID for our tunnel, run the following command:

```powershell
cloudflared tunnel token my-gcp-tunnel
cloudflared tunnel info my-gcp-tunnel
```

Update the startup script with the following content:

```bash
#!/bin/bash
set -e
echo "=== Startup script started at $(date) ==="
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared-linux-amd64.deb
mkdir -p /etc/cloudflared/
cat > /etc/cloudflared/config.yml << EOF
tunnel: <MY_TUNNEL_ID>
credentials-file: /root/.cloudflared/<MY_TUNNEL_ID>.json
ingress:
  - hostname: nginx.<MY_DOMAIN>
    service: http://127.0.0.1:80
  - service: http_status:404
EOF
cloudflared service install <MY_TUNNEL_TOKEN> --config /etc/cloudflared/config.yml
systemctl start cloudflared
systemctl enable cloudflared
echo "=== Startup script completed at $(date) ==="
```

This script will now perform the following actions:

* Install NGINX.
    
* Download and install the cloudflared daemon.
    
* Configure and start the cloudflared service using our tunnel token. This will connect our VM to the Cloudflare network and route traffic from our chosen hostname to the local NGINX server on port `80`.
    

## Deploy

Now, we can apply the changes using Infrastructure Manager:

```powershell
gcloud infra-manager deployments apply my-first-deployment --location=us-central1 --local-source=./terraform --input-values=project_id=<MY_PROJECT_ID> --service-account=projects/<MY_PROJECT_ID>/serviceAccounts/infra-manager-sa@<MY_PROJECT_ID>.iam.gserviceaccount.com
```

Once the deployment is complete, our NGINX server will be accessible at the hostname you configured (nginx.&lt;MY\_DOMAIN&gt; in our example), secured behind the Cloudflare network.

## Clean Up

To remove all the resources created, we can run the same cleanup command as in the previous article:

```powershell
gcloud infra-manager deployments delete my-deployment --location=us-central1
```

To remove the Cloudflare tunnel, we can run:

```powershell
cloudflared tunnel delete my-gcp-tunnel
```

By using Cloudflare Tunnel, we've greatly enhanced the security of our deployment, showing a more production-ready way to expose services running on GCP. You can find all the code [here](https://github.com/raulnq/gcp-infrastructure-manage/tree/cloudflare-tunnel). Thanks, and happy coding.