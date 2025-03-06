# BrightPath Terraform Infrastructure Plan

## Overview

This document outlines the strategy for using Terraform to provision and manage the Digital Ocean infrastructure for the BrightPath homeschooling scheduling platform. Infrastructure as Code (IaC) with Terraform ensures consistent, reproducible, and version-controlled infrastructure deployments across all environments.

## Infrastructure as Code Strategy

### Key Principles

1. **Infrastructure as Code**: All infrastructure is defined as code in Terraform configurations.
2. **Version Control**: All Terraform code is stored in Git alongside application code.
3. **Environment Parity**: Development, staging, and production environments share the same Terraform configurations with environment-specific variables.
4. **Modularity**: Infrastructure components are organized into reusable Terraform modules.
5. **Least Privilege**: Resources are provisioned with minimal required permissions.
6. **Immutability**: Infrastructure changes are applied by creating new resources rather than modifying existing ones when possible.

### Benefits

- **Consistency**: Eliminates environment drift and "works on my machine" problems
- **Reproducibility**: Enables reliable recreation of environments
- **Auditability**: Provides a clear history of infrastructure changes
- **Collaboration**: Allows team members to review infrastructure changes
- **Automation**: Facilitates CI/CD integration for infrastructure deployment

## Terraform Structure

### Repository Organization

```
infrastructure/
├── environments/
│   ├── development/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── droplets/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── object_storage/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── .gitignore
└── README.md
```

### Module Design

Each module encapsulates a specific infrastructure component and follows these design principles:

1. **Single Responsibility**: Each module manages one type of infrastructure component.
2. **Configurable**: Modules accept variables for customization.
3. **Reusable**: Modules can be used across different environments.
4. **Testable**: Modules can be tested independently.
5. **Documented**: Each module includes documentation on usage and variables.

## Digital Ocean Resources

The following Digital Ocean resources will be managed with Terraform:

### 1. Networking

```hcl
# modules/networking/main.tf
resource "digitalocean_vpc" "brightpath_vpc" {
  name        = "${var.environment}-vpc"
  region      = var.region
  description = "VPC for BrightPath ${var.environment} environment"
}

resource "digitalocean_firewall" "web" {
  name = "${var.environment}-web-firewall"

  # Allow HTTP and HTTPS from anywhere
  inbound_rule {
    protocol         = "tcp"
    port_range       = "80"
    source_addresses = ["0.0.0.0/0", "::/0"]
  }

  inbound_rule {
    protocol         = "tcp"
    port_range       = "443"
    source_addresses = ["0.0.0.0/0", "::/0"]
  }

  # Allow SSH from specific IPs
  inbound_rule {
    protocol         = "tcp"
    port_range       = "22"
    source_addresses = var.allowed_ssh_ips
  }

  # Allow all outbound traffic
  outbound_rule {
    protocol              = "tcp"
    port_range            = "1-65535"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }

  outbound_rule {
    protocol              = "udp"
    port_range            = "1-65535"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }

  outbound_rule {
    protocol              = "icmp"
    destination_addresses = ["0.0.0.0/0", "::/0"]
  }
}
```

### 2. Droplets and Load Balancer

```hcl
# modules/droplets/main.tf
resource "digitalocean_droplet" "web" {
  name     = "${var.environment}-web-${count.index + 1}"
  size     = var.droplet_size
  image    = var.droplet_image
  region   = var.region
  vpc_uuid = var.vpc_id
  ssh_keys = var.ssh_key_ids
  count    = var.web_droplet_count
  tags     = ["${var.environment}", "web", "brightpath"]

  user_data = templatefile("${path.module}/templates/cloud-init.yml", {
    app_name = "brightpath"
    environment = var.environment
    docker_compose_version = var.docker_compose_version
  })
}

resource "digitalocean_droplet" "worker" {
  name     = "${var.environment}-worker-${count.index + 1}"
  size     = var.droplet_size
  image    = var.droplet_image
  region   = var.region
  vpc_uuid = var.vpc_id
  ssh_keys = var.ssh_key_ids
  count    = var.worker_droplet_count
  tags     = ["${var.environment}", "worker", "brightpath"]

  user_data = templatefile("${path.module}/templates/cloud-init.yml", {
    app_name = "brightpath"
    environment = var.environment
    docker_compose_version = var.docker_compose_version
  })
}

resource "digitalocean_loadbalancer" "web" {
  name   = "${var.environment}-loadbalancer"
  region = var.region
  vpc_uuid = var.vpc_id

  forwarding_rule {
    entry_port     = 80
    entry_protocol = "http"
    target_port     = 8000
    target_protocol = "http"
  }

  forwarding_rule {
    entry_port     = 443
    entry_protocol = "https"
    target_port     = 8000
    target_protocol = "http"
    certificate_id = var.certificate_id
  }

  healthcheck {
    port     = 8000
    protocol = "http"
    path     = "/health/"
  }

  droplet_ids = digitalocean_droplet.web[*].id
}

# Cloud-init template for Droplet provisioning
resource "local_file" "cloud_init_template" {
  content = <<-EOT
#cloud-config
package_update: true
package_upgrade: true

packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg-agent
  - software-properties-common
  - python3-pip
  - ufw

runcmd:
  # Install Docker
  - curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  - add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  - apt-get update -y
  - apt-get install -y docker-ce docker-ce-cli containerd.io
  - systemctl enable docker
  - systemctl start docker

  # Install Docker Compose
  - curl -L "https://github.com/docker/compose/releases/download/${docker_compose_version}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  - chmod +x /usr/local/bin/docker-compose

  # Configure firewall
  - ufw allow OpenSSH
  - ufw allow 80/tcp
  - ufw allow 443/tcp
  - ufw --force enable

  # Create app directory
  - mkdir -p /opt/${app_name}
  - chmod 755 /opt/${app_name}

  # Set up Docker log rotation
  - echo '{"log-driver":"json-file","log-opts":{"max-size":"10m","max-file":"3"}}' > /etc/docker/daemon.json
  - systemctl restart docker
EOT

  filename = "${path.module}/templates/cloud-init.yml"
}
```

### 3. Database

```hcl
# modules/database/main.tf
resource "digitalocean_database_cluster" "postgres" {
  name       = "${var.environment}-postgres-cluster"
  engine     = "pg"
  version    = var.postgres_version
  size       = var.db_size
  region     = var.region
  node_count = var.environment == "production" ? 2 : 1
  private_network_uuid = var.vpc_id

  maintenance_window {
    day  = "sunday"
    hour = "02:00:00"
  }
}

resource "digitalocean_database_db" "brightpath_db" {
  cluster_id = digitalocean_database_cluster.postgres.id
  name       = "brightpath"
}

resource "digitalocean_database_user" "brightpath_user" {
  cluster_id = digitalocean_database_cluster.postgres.id
  name       = "brightpath_app"
}

resource "digitalocean_database_firewall" "postgres_fw" {
  cluster_id = digitalocean_database_cluster.postgres.id

  # Allow access from web and worker droplets
  dynamic "rule" {
    for_each = concat(
      digitalocean_droplet.web[*].id,
      digitalocean_droplet.worker[*].id
    )

    content {
      type  = "droplet"
      value = rule.value
    }
  }

  # For development, allow specific IPs
  dynamic "rule" {
    for_each = var.environment == "development" ? var.allowed_db_ips : []
    content {
      type  = "ip_addr"
      value = rule.value
    }
  }
}
```

### 4. Object Storage

```hcl
# modules/object_storage/main.tf
resource "digitalocean_spaces_bucket" "static_assets" {
  name   = "${var.environment}-brightpath-assets"
  region = var.region
  acl    = "public-read"

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET"]
    allowed_origins = var.allowed_origins
    max_age_seconds = 3600
  }
}

resource "digitalocean_spaces_bucket" "user_uploads" {
  name   = "${var.environment}-brightpath-uploads"
  region = var.region
  acl    = "private"

  versioning {
    enabled = true
  }
}

resource "digitalocean_cdn" "static_assets_cdn" {
  origin           = digitalocean_spaces_bucket.static_assets.bucket_domain_name
  custom_domain    = var.environment == "production" ? "assets.brightpath.com" : null
  certificate_name = var.environment == "production" ? "brightpath-assets-cert" : null
  ttl              = 3600
}
```

### 5. Monitoring

```hcl
# modules/monitoring/main.tf
resource "digitalocean_monitor_alert" "high_cpu" {
  alerts {
    email = var.alert_emails
    slack {
      channel = var.slack_channel
      url     = var.slack_webhook_url
    }
  }

  window      = "5m"
  type        = "v1/insights/droplet/cpu"
  compare     = "GreaterThan"
  value       = 70
  enabled     = true
  entities    = var.droplet_ids
  description = "CPU usage is above 70%"
}

resource "digitalocean_monitor_alert" "high_memory" {
  alerts {
    email = var.alert_emails
    slack {
      channel = var.slack_channel
      url     = var.slack_webhook_url
    }
  }

  window      = "5m"
  type        = "v1/insights/droplet/memory_utilization_percent"
  compare     = "GreaterThan"
  value       = 80
  enabled     = true
  entities    = var.droplet_ids
  description = "Memory usage is above 80%"
}

resource "digitalocean_monitor_alert" "disk_space" {
  alerts {
    email = var.alert_emails
    slack {
      channel = var.slack_channel
      url     = var.slack_webhook_url
    }
  }

  window      = "5m"
  type        = "v1/insights/droplet/disk_utilization_percent"
  compare     = "GreaterThan"
  value       = 85
  enabled     = true
  entities    = var.droplet_ids
  description = "Disk usage is above 85%"
}
```

## Environment Configuration

### Development Environment

```hcl
# environments/development/main.tf
provider "digitalocean" {
  token = var.do_token
}

terraform {
  backend "s3" {
    bucket                      = "brightpath-terraform-state"
    key                         = "development/terraform.tfstate"
    endpoint                    = "nyc3.digitaloceanspaces.com"
    region                      = "us-east-1"  # Placeholder, DO Spaces uses different regions
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}

module "networking" {
  source = "../../modules/networking"

  environment     = "development"
  region          = var.region
  allowed_ssh_ips = var.allowed_ssh_ips
}

module "droplets" {
  source = "../../modules/droplets"

  environment           = "development"
  region                = var.region
  vpc_id                = module.networking.vpc_id
  droplet_image         = "ubuntu-20-04-x64"
  droplet_size          = "s-2vcpu-4gb"
  web_droplet_count     = 1
  worker_droplet_count  = 1
  ssh_key_ids           = var.ssh_key_ids
  docker_compose_version = "1.29.2"
  certificate_id        = var.certificate_id
}

module "database" {
  source = "../../modules/database"

  environment          = "development"
  region               = var.region
  vpc_id               = module.networking.vpc_id
  postgres_version     = var.postgres_version
  db_size              = "db-s-1vcpu-1gb"
  web_droplet_ids      = module.droplets.web_droplet_ids
  worker_droplet_ids   = module.droplets.worker_droplet_ids
  allowed_db_ips       = var.allowed_db_ips
}

module "object_storage" {
  source = "../../modules/object_storage"

  environment     = "development"
  region          = var.region
  allowed_origins = ["http://localhost:3000", "https://dev.brightpath.com"]
}

module "monitoring" {
  source = "../../modules/monitoring"

  alert_emails      = var.alert_emails
  slack_channel     = var.slack_channel
  slack_webhook_url = var.slack_webhook_url
  droplet_ids       = concat(
    module.droplets.web_droplet_ids,
    module.droplets.worker_droplet_ids
  )
}
```

### Production Environment

```hcl
# environments/production/main.tf
provider "digitalocean" {
  token = var.do_token
}

terraform {
  backend "s3" {
    bucket                      = "brightpath-terraform-state"
    key                         = "production/terraform.tfstate"
    endpoint                    = "nyc3.digitaloceanspaces.com"
    region                      = "us-east-1"  # Placeholder, DO Spaces uses different regions
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}

module "networking" {
  source = "../../modules/networking"

  environment     = "production"
  region          = var.region
  allowed_ssh_ips = var.allowed_ssh_ips
}

module "droplets" {
  source = "../../modules/droplets"

  environment           = "production"
  region                = var.region
  vpc_id                = module.networking.vpc_id
  droplet_image         = "ubuntu-20-04-x64"
  droplet_size          = "s-4vcpu-8gb"
  web_droplet_count     = 3
  worker_droplet_count  = 2
  ssh_key_ids           = var.ssh_key_ids
  docker_compose_version = "1.29.2"
  certificate_id        = var.certificate_id
}

module "database" {
  source = "../../modules/database"

  environment          = "production"
  region               = var.region
  vpc_id               = module.networking.vpc_id
  postgres_version     = var.postgres_version
  db_size              = "db-s-2vcpu-4gb"
  web_droplet_ids      = module.droplets.web_droplet_ids
  worker_droplet_ids   = module.droplets.worker_droplet_ids
  allowed_db_ips       = []  # No direct access in production
}

module "object_storage" {
  source = "../../modules/object_storage"

  environment     = "production"
  region          = var.region
  allowed_origins = ["https://brightpath.com", "https://www.brightpath.com"]
}

module "monitoring" {
  source = "../../modules/monitoring"

  alert_emails      = var.alert_emails
  slack_channel     = var.slack_channel
  slack_webhook_url = var.slack_webhook_url
  droplet_ids       = concat(
    module.droplets.web_droplet_ids,
    module.droplets.worker_droplet_ids
  )
}
```

## State Management

### Remote State Storage

Terraform state will be stored in Digital Ocean Spaces (S3-compatible storage) to enable collaboration and state locking:

```hcl
# Setup for state bucket (run once manually)
resource "digitalocean_spaces_bucket" "terraform_state" {
  name   = "brightpath-terraform-state"
  region = "nyc3"
  acl    = "private"

  versioning {
    enabled = true
  }
}
```

### State Locking

To prevent concurrent modifications to the infrastructure, state locking will be implemented using a DynamoDB-compatible service or by ensuring that infrastructure changes are only applied through the CI/CD pipeline.

### State Organization

- Each environment (development, staging, production) has its own state file
- State files are organized by environment in the Spaces bucket
- Remote state data is shared between environments using Terraform data sources

## CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [ main ]
    paths:
      - 'infrastructure/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'infrastructure/**'

jobs:
  validate:
    name: Validate
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: |
          cd infrastructure/environments/development
          terraform init -backend=false

      - name: Terraform Validate
        run: |
          cd infrastructure/environments/development
          terraform validate

  plan:
    name: Plan
    needs: validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0

      - name: Terraform Init
        run: |
          cd infrastructure/environments/development
          terraform init \
            -backend-config="access_key=${{ secrets.DO_SPACES_ACCESS_KEY }}" \
            -backend-config="secret_key=${{ secrets.DO_SPACES_SECRET_KEY }}"
        env:
          DIGITALOCEAN_TOKEN: ${{ secrets.DIGITALOCEAN_TOKEN }}

      - name: Terraform Plan
        run: |
          cd infrastructure/environments/development
          terraform plan -var "do_token=${{ secrets.DIGITALOCEAN_TOKEN }}" -out=tfplan
        env:
          DIGITALOCEAN_TOKEN: ${{ secrets.DIGITALOCEAN_TOKEN }}

      - name: Upload Plan
        uses: actions/upload-artifact@v2
        with:
          name: tfplan
          path: infrastructure/environments/development/tfplan

  apply:
    name: Apply
    needs: plan
    runs-on: ubuntu-latest
    if: github.event.pull_request.merged == true
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: 1.0.0

      - name: Download Plan
        uses: actions/download-artifact@v2
        with:
          name: tfplan
          path: infrastructure/environments/development

      - name: Terraform Init
        run: |
          cd infrastructure/environments/development
          terraform init \
            -backend-config="access_key=${{ secrets.DO_SPACES_ACCESS_KEY }}" \
            -backend-config="secret_key=${{ secrets.DO_SPACES_SECRET_KEY }}"
        env:
          DIGITALOCEAN_TOKEN: ${{ secrets.DIGITALOCEAN_TOKEN }}

      - name: Terraform Apply
        run: |
          cd infrastructure/environments/development
          terraform apply tfplan
        env:
          DIGITALOCEAN_TOKEN: ${{ secrets.DIGITALOCEAN_TOKEN }}
```

### Deployment Process

1. **Development Changes**:
   - Developer creates a branch and makes infrastructure changes
   - Pull request triggers Terraform validation and plan
   - Team reviews the plan in the PR
   - Merging the PR triggers Terraform apply

2. **Production Deployment**:
   - Changes are first applied to staging environment
   - After validation, a PR is created for production
   - Team reviews the production plan
   - Approval and merge triggers production apply

## Security Considerations

### Secrets Management

- Digital Ocean API tokens and other secrets are stored in GitHub Secrets
- Sensitive outputs are marked as sensitive in Terraform
- No secrets are hardcoded in Terraform configurations

### Access Control

- IAM roles and policies follow the principle of least privilege
- Network access is restricted to necessary services
- Firewall rules limit access to resources

### Encryption

- All data at rest is encrypted
- All data in transit uses TLS
- Database connections use SSL

## Disaster Recovery

### Backup Strategy

- Database backups are automated through Digital Ocean
- Terraform state is versioned in Spaces
- Critical application data is backed up regularly

### Recovery Process

1. **State Corruption**:
   - Revert to previous state version in Spaces
   - Run targeted apply to fix specific resources

2. **Complete Recreation**:
   - Use Terraform to recreate infrastructure in new region
   - Restore data from backups

## Cost Optimization

### Resource Sizing

- Development environment uses smaller instances
- Production uses right-sized resources based on load testing
- Auto-scaling is configured to optimize costs

### Resource Scheduling

- Non-production environments can be scheduled to shut down during off-hours
- Spot instances are used where appropriate

## Implementation Timeline

### Phase 1: Foundation (Weeks 1-2)

- Set up Terraform repository structure
- Create networking and basic infrastructure modules
- Implement state management in Spaces
- Set up CI/CD for Terraform

### Phase 2: Core Infrastructure (Weeks 3-4)

- Implement Droplets module
- Set up database module
- Configure object storage
- Implement monitoring

### Phase 3: Environment Configuration (Weeks 5-6)

- Set up development environment
- Configure staging environment
- Implement production environment
- Test environment promotion

### Phase 4: Optimization and Documentation (Weeks 7-8)

- Optimize resource usage
- Implement cost controls
- Complete documentation
- Train team on infrastructure management

## Conclusion

This Terraform infrastructure plan provides a comprehensive approach to managing Digital Ocean infrastructure for the BrightPath platform. By implementing infrastructure as code with Terraform, we ensure consistent, reproducible, and version-controlled infrastructure deployments across all environments.

The modular structure allows for reuse of components and easy adaptation to changing requirements. The CI/CD integration ensures that infrastructure changes follow the same review and approval process as application code changes.

By following this plan, the BrightPath team can efficiently manage their Digital Ocean infrastructure while maintaining security, reliability, and cost-effectiveness.