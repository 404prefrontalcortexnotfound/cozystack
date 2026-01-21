# Future Proofing: Terraform Infrastructure as Code

## Objective

Manage Hetzner control nodes and local worker (heimdall) with unified Terraform configuration using the Talos provider.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Terraform Configuration (Single Codebase)              │
│  ├── providers.tf                                       │
│  ├── hetzner-nodes.tf (3x control-plane)                │
│  ├── local-nodes.tf (1x heimdall control+worker)        │
│  └── outputs.tf (kubeconfig, talosconfig)               │
│                                                         │
│  Managed Resources:                                     │
│  ├── Hetzner Cloud servers (via hetzner provider)       │
│  ├── Talos machine configs (via talos provider)         │
│  ├── Bootstrap etcd cluster                             │
│  └── Generate kubeconfig/talosconfig                    │
└─────────────────────────────────────────────────────────┘
```

## Benefits

**Infrastructure as Code**: Version-controlled cluster definition
**Reproducibility**: Rebuild entire cluster from code
**Documentation**: Configuration self-documents infrastructure
**Testing**: Spin up staging clusters for validation
**Disaster Recovery**: Rapid cluster recreation

## Terraform Providers

```hcl
terraform {
  required_providers {
    hcloud = {
      source  = "hetznercloud/hcloud"
      version = "~> 1.45"
    }
    talos = {
      source  = "siderolabs/talos"
      version = "~> 0.7"
    }
  }
}

provider "hcloud" {
  token = var.hcloud_token
}

provider "talos" {}
```

## Directory Structure

```
terraform/
├── providers.tf              # Provider configuration
├── variables.tf              # Input variables
├── hetzner-control-nodes.tf  # 3x Hetzner CPX41 control nodes
├── local-worker.tf           # heimdall configuration
├── talos-config.tf           # Talos machine configs
├── bootstrap.tf              # Cluster bootstrap
├── outputs.tf                # kubeconfig, talosconfig outputs
├── terraform.tfvars.example  # Example variables file
└── README.md                 # Usage documentation
```

## Key Resources

### Hetzner Control Nodes

```hcl
resource "hcloud_server" "control" {
  count       = 3
  name        = "cozy-ctrl-${count.index + 1}"
  server_type = "cpx41"  # 8 vCPU, 16GB RAM
  location    = "hel1"   # Helsinki
  image       = "ubuntu-24.04"

  user_data = templatefile("${path.module}/cloud-init-talos.yaml", {
    talos_version = var.talos_version
  })

  labels = {
    role    = "control-plane"
    cluster = "cozystack"
  }
}
```

### Talos Machine Configurations

```hcl
resource "talos_machine_configuration" "control" {
  cluster_name = "cozystack"
  machine_type = "controlplane"

  cluster_endpoint = "https://${hcloud_server.control[0].ipv4_address}:6443"

  machine_secrets = talos_machine_secrets.this.machine_secrets

  config_patches = [
    yamlencode({
      machine = {
        kubelet = {
          extraArgs = {
            rotate-server-certificates = "true"
          }
        }
      }
      cluster = {
        network = {
          cni = {
            name = "none"  # Cozystack manages CNI
          }
        }
      }
    })
  ]
}
```

### Local Worker (heimdall)

```hcl
resource "talos_machine_configuration" "local_worker" {
  cluster_name = "cozystack"
  machine_type = "worker"

  cluster_endpoint = "https://${hcloud_server.control[0].ipv4_address}:6443"

  machine_secrets = talos_machine_secrets.this.machine_secrets

  config_patches = [
    yamlencode({
      machine = {
        network = {
          interfaces = [{
            interface = "tailscale0"
            addresses = [var.heimdall_tailscale_ip]
          }]
        }
        sysctls = {
          "net.core.rmem_max" = "2500000"  # SeaweedFS tuning
        }
        kubelet = {
          extraMounts = [{
            destination = "/var/mnt/seaweedfs"
            type        = "bind"
            source      = "/mnt/seaweedfs"
            options     = ["bind", "rw"]
          }]
        }
      }
    })
  ]
}
```

## Workflow

### Initial Deployment

```bash
cd terraform/

# Initialize Terraform
terraform init

# Review planned changes
terraform plan

# Apply configuration
terraform apply

# Extract kubeconfig
terraform output -raw kubeconfig > ~/.kube/cozy-stack-kubeconfig

# Extract talosconfig
terraform output -raw talosconfig > ~/.talos/cozy-stack-talosconfig
```

### Adding Nodes

1. Modify `hetzner-control-nodes.tf` to increase count
2. Run `terraform plan` to preview changes
3. Run `terraform apply` to provision new node
4. Talos automatically joins cluster

### Removing Nodes

1. Decrease count in configuration
2. Run `terraform apply`
3. Terraform drains and destroys node

### Disaster Recovery

```bash
# Cluster lost, rebuild from code
cd terraform/
terraform destroy  # Clean slate
terraform apply    # Rebuild identical cluster

# Restore application state from backups
kubectl apply -f manifests/
```

## Variables Configuration

`terraform.tfvars`:

```hcl
hcloud_token         = "xxxxxxxxxxxxx"  # Store in Bitwarden
talos_version        = "v1.10.5"
cluster_endpoint     = "cozy.homi.zone"
heimdall_tailscale_ip = "100.x.x.x/32"

control_nodes = {
  count    = 3
  type     = "cpx41"
  location = "hel1"
}
```

## State Management

**Remote State**: Store Terraform state in S3-compatible backend (SeaweedFS)

```hcl
terraform {
  backend "s3" {
    bucket   = "terraform-state"
    key      = "cozystack/terraform.tfstate"
    endpoint = "https://s3.cozy.homi.zone"

    skip_credentials_validation = true
    skip_region_validation      = true
  }
}
```

**State Locking**: Prevent concurrent modifications

## Security Considerations

**Secrets Management**:
- Hetzner API token → Bitwarden Secrets Manager
- Talos secrets → Generated by provider, stored in state
- Never commit `terraform.tfvars` to git

**State Encryption**: Enable S3 backend encryption

## Integration with Cozystack

Terraform provisions bare Talos cluster. Post-deployment:

1. Install Cozystack via Helm (automated in bootstrap.tf)
2. Apply GitOps manifests for applications
3. Cozystack manages application lifecycle

## Future Enhancements

- Add Vault integration for secret management
- Implement blue/green deployment for node rotation
- Add monitoring for Terraform drift detection
- Create CI/CD pipeline for automated plan/apply
- Add cost estimation via Infracost
