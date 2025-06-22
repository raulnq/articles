---
title: "Terraform as a Service with Google Cloud's Infrastructure Manager"
datePublished: Sun Jun 22 2025 01:57:10 GMT+0000 (Coordinated Universal Time)
cuid: cmc70p2f4000602jr8fnscldg
slug: terraform-as-a-service-with-google-clouds-infrastructure-manager
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1750448784052/dee2e370-e84d-42e2-9ee5-056d86e62102.png
tags: terraform, gcp, iac

---

GCP (Google Cloud Platform) [Infrastructure Manager](https://cloud.google.com/infrastructure-manager/docs/overview) is a fully managed infrastructure-as-code (IaC) service that allows us to provision, manage, and orchestrate Google Cloud resources using Terraform. It provides a managed way to use [Terraform with GCP,](https://registry.terraform.io/providers/hashicorp/google/latest/docs) eliminating the need for us to manage the state ourselves. Think of it as Terraform-as-a-Service living directly within our GCP project. Instead of running `terraform apply` on our environment and worrying about where to store the state file, we hand our Terraform code to Infrastructure Manager, and it handles the rest.

GCP Infrastructure Manager doesn't have direct charges. Instead, it charges for Cloud Build execution and Cloud Storage. [Cloud Build](https://cloud.google.com/build/docs/overview) includes 2,500 free build-minutes each month before charging begins and [Cloud Storage](https://cloud.google.com/storage/docs/introduction) has a free tier with 5 GB of standard storage per month.

Let's create a practical example that deploys a free-tier VM with NGINX exposed to the internet, executed locally using Infrastructure Manager.

## Pre-requisites

* A [Google Cloud Project](https://developers.google.com/workspace/guides/create-project)**.**
    
* [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) installed and authenticated.
    

## APIs

We need to enable the Infrastructure Manager API, the Compute Engine API, and the Cloud Storage API:

```powershell
gcloud services enable config.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable storage.googleapis.com
```

## Terraform

Create a `terraform/main.tf` file with the following content:

```javascript
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
  tags = ["http-server"]
  metadata = {
    startup-script = file("${path.module}/startup-script.sh")
  }
}

resource "google_compute_firewall" "allow_http" {
  project = var.project_id
  name    = "allow-http-ingress"
  network = "default"
  allow {
    protocol = "tcp"
    ports    = ["80"]
  }
  target_tags = ["http-server"]
  source_ranges = ["0.0.0.0/0"]
}

variable "project_id" {
  type        = string
  description = "The GCP project ID to deploy resources into."
}

output "vm_external_ip" {
  description = "The external IP address of the web server VM."
  value       = google_compute_instance.free_tier_vm.network_interface[0].access_config[0].nat_ip
}
```

This Terraform script creates a basic web server using free-tier resources. Here's what it does:

* [`google_compute_instance`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_instance): Creates an `e2-micro` VM instance (free-tier eligible) deployed in `us-central1-a` (free-tier region) . The `boot_disk` block defines the main disk that contains the operating system (specified by the `image` property) used to start the VM instance. The `network_interface` block automatically assigns an internal (private) IP address in the `default` VPC network. The `access_config` block instructs GCP to assign an external (public) IP address. By default, this IP address is ephemeral (changes when VM restarts). The `tags` property is used by the firewall rule to identify the VM instance. The `startup_script` property defines a script that runs automatically when the VM starts up for the first time.
    
* [`google_compute_firewall`](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/compute_firewall): Creates a rule named `allow-http-ingress` that opens `TCP` port `80` (HTTP) for incoming traffic, allowing access from anywhere on the internet (`0.0.0.0/0`). This rule applies only to instances tagged with `http-server`.
    

Create the `terraform/startup-script.sh` with the following content:

```bash
#!/bin/bash
set -e
echo "=== Startup script started at $(date) ==="
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
echo "=== Startup script completed at $(date) ==="
```

> Windows line endings could produce errors. Ensure the file uses Unix line endings (`LF`, not `CRLF`).

## Deploy

Infrastructure Manager requires a service account with specific IAM roles:

```powershell
gcloud iam service-accounts create infra-manager-sa --display-name=infra-manager-sa --project=<MY_PROJECT_ID>
gcloud projects add-iam-policy-binding <MY_PROJECT_ID> --member="serviceAccount:infra-manager-sa@<MY_PROJECT_ID>.iam.gserviceaccount.com" --role="roles/admin"
```

Now, we can use the following command to deploy our VM:

```powershell
gcloud infra-manager deployments apply my-first-deployment --location=us-central1 --local-source=./terraform --input-values=project_id=<MY_PROJECT_ID> --service-account=projects/<MY_PROJECT_ID>/serviceAccounts/infra-manager-sa@<MY_PROJECT_ID>.iam.gserviceaccount.com
```

After the command finishes, we can check the public IP of the VM with the following command:

```powershell
gcloud compute instances describe my-first-vm --zone=us-central1-a --format="value(networkInterfaces[0].accessConfigs[0].natIP)"
```

## Clean Up

To avoid any charges and clean up resources, run:

```powershell
gcloud infra-manager deployments delete my-deployment --location=us-central1
```

> The [official documentation](https://cloud.google.com/sdk/gcloud/reference/infra-manager/deployments) offers more details about the commands used.

You can find all the code [here](https://github.com/raulnq/gcp-infrastructure-manage). Thanks, and happy coding.