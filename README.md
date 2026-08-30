# GCP-Security-Labs-Infrastructure-Automation

Here is a comprehensive **`README.md`** file that contains both the **manual console steps** and the **Terraform configuration scripts** tailored specifically for your GitHub repository based on your labs.

---

### File 1: `README.md`

```markdown
# GCP Cloud Security Infrastructure & Vulnerability Management

This repository contains Infrastructure as Code (Terraform) configurations, network architecture definitions, and comprehensive manual operational guides for deploying, securing, and hardening infrastructure on Google Cloud Platform (GCP).

---

## 📁 Repository Structure
```text
├── README.md             # Complete manual steps and documentation
├── main.tf               # Terraform infrastructure configuration
├── variables.tf          # Terraform input variables
└── outputs.tf            # Terraform deployment outputs

```

---

## 🛠️ Part 1: Manual Step-by-Step Guide

### Task 1: Create a Custom VPC Network & Virtual Machine

1. **Create a Static IP Address:**
* Open **Cloud Shell** in your Google Cloud console.


* Run the following command to reserve a static IP:
```bash
gcloud compute addresses create xss-test-ip-address --region="us-west1"

```




2. **Launch the Virtual Machine Instance:**
* Run the following `gcloud` command to create a VM equipped with a startup script that provisions Python and Flask:
```bash
gcloud compute instances create xss-test-vm-instance --address=xss-test-ip-address --no-service-account --no-scopes --machine-type=e2-micro --zone="us-west1-a" --metadata=startup-script='apt-get update; apt-get install -y python3-flask'

```





### Task 2: Configure Application Network & Ingress Access

1. **Create a Firewall Rule for Port 8080:**
* Run the following command to allow web security scanners and visitors to hit your test port:
```bash
gcloud compute firewall-rules create enable-wss-scan --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:8080 --source-ranges=0.0.0.0/0

```




2. **Deploy Code via SSH:**
* Go to **Compute Engine** $\rightarrow$ **VM instances**.


* Click the **SSH** button next to `xss-test-vm-instance`.


* Download and extract the vulnerable application code:


```bash
gsutil cp gs://cloud-training/GCPSEC-ScannerAppEngine/flask_code.tar . && tar xvf flask_code.tar
python3 app.py

```





### Task 3: Simulating and Scanning Vulnerabilities (XSS)

1. Navigate to `http://<YOUR_EXTERNAL_IP>:8080` in an incognito window.
2. Inject a test script payload into the input field to confirm the Cross-Site Scripting (XSS) vulnerability:
```html
<script>alert('This is an XSS Injection to demonstrate one of OWASP vulnerabilities')</script>

```


3. Enable the **Web Security Scanner API** via APIs & Services, then configure and execute a scan in **Security** $\rightarrow$ **Web Security Scanner**.



### Task 4: Remediating Vulnerabilities

1. Stop the application in your SSH terminal (`Ctrl + C`).


2. Open the application script using `nano app.py`.


3. Modify the code to uncomment the sanitization/escaping lines and comment out the unsafe raw input lines:


```python
output_string = "".join([html_escape_table.get(c, c) for c in input_string])
# output_string = input_string

```


4. Save and restart the application (`python3 app.py`), then re-run your web security scan to ensure clean results.



---

## ⚙️ Part 2: Terraform Infrastructure as Code

If you prefer to deploy your base architecture via Terraform rather than manually, use the configuration files below.

### `main.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

# 1. Custom VPC Network
resource "google_compute_network" "vpc_network" {
  name                    = var.network_name
  auto_create_subnetworks = false
  mtu                     = 1460
}

# 2. Custom Subnetwork
resource "google_compute_subnetwork" "default" {
  name          = var.subnet_name
  ip_cidr_range = var.subnet_cidr
  region        = var.region
  network       = google_compute_network.vpc_network.id
}

# 3. Firewall Rule for App Traffic (Port 8080 & SSH)
resource "google_compute_firewall" "allow_app_traffic" {
  name    = "allow-app-traffic"
  network = google_compute_network.vpc_network.name

  allow {
    protocol = "tcp"
    ports    = ["22", "8080"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["http-server", "xss-test"]
}

# 4. Compute Engine Instance
resource "google_compute_instance" "default" {
  name         = "xss-test-vm-instance"
  machine_type = "e2-micro"
  zone         = var.zone
  tags         = ["http-server", "xss-test"]

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  metadata_startup_script = <<-EOT
    sudo apt-get update;
    sudo apt-get install -yq build-essential python3-pip rsync;
    pip install flask;
  EOT

  network_interface {
    subnetwork = google_compute_subnetwork.default.id
    access_config {
      // Ephemeral public IP
    }
  }
}

```

### `variables.tf`

```hcl
variable "project_id" {
  type        = string
  description = "Your GCP Project ID"
}

variable "region" {
  type        = string
  default     = "us-west1"
}

variable "zone" {
  type        = string
  default     = "us-west1-a"
}

variable "network_name" {
  type        = string
  default     = "my-custom-mode-network"
}

variable "subnet_name" {
  type        = string
  default     = "my-custom-subnet"
}

variable "subnet_cidr" {
  type        = string
  default     = "10.0.1.0/24"
}

```

### `outputs.tf`

```hcl
output "vm_external_ip" {
  value       = google_compute_instance.default.network_interface[0].access_config[0].nat_ip
  description = "External IP address of the test instance."
}

output "vpc_network_id" {
  value       = google_compute_network.vpc_network.id
  description = "The ID of the custom VPC network."
}

```

---

## 🚀 Quick Start Deployment

1. Clone this repository to your local machine or Cloud Shell.


2. Initialize and apply the Terraform configuration:
```bash
terraform init
terraform apply -var="project_id=YOUR_GCP_PROJECT_ID"

```



```

```
