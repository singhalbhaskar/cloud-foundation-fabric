# GKE nodepool module

This module allows simplified creation and management of individual GKE nodepools, setting sensible defaults (eg a service account is created for nodes if none is set) and allowing for less verbose usage in most use cases.

## Example usage

### Module defaults

If no specific node configuration is set via variables, the module uses the provider's defaults only setting OAuth scopes to a minimal working set and the node machine type to `n1-standard-1`. The service account set by the provider in this case is the GCE default service account.

```hcl
module "cluster-1-nodepool-1" {
  source       = "./fabric/modules/gke-nodepool"
  project_id   = "myproject"
  cluster_name = "cluster-1"
  location     = "europe-west1-b"
  name         = "nodepool-1"
}
# tftest modules=1 resources=1 inventory=basic.yaml
```

### Internally managed service account

There are three different approaches to defining the nodes service account, all depending on the `service_account` variable where the `create` attribute controls creation of a new service account by this module, and the `email` attribute controls the actual service account to use.

If you create a new service account, its resource and email (in both plain and IAM formats) are then available in outputs to reference it in other modules or resources.

#### GCE default service account

To use the GCE default service account, you can ignore the variable which is equivalent to `{ create = null, email = null }`. This is what the first example of this document does.

#### Externally defined service account

To use an existing service account, pass in just the `email` attribute. If you do this, will most likely want to use the `cloud-platform` scope.

```hcl
module "cluster-1-nodepool-1" {
  source       = "./fabric/modules/gke-nodepool"
  project_id   = "myproject"
  cluster_name = "cluster-1"
  location     = "europe-west1-b"
  name         = "nodepool-1"
  service_account = {
    email        = "foo-bar@myproject.iam.gserviceaccount.com"
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}
# tftest modules=1 resources=1 inventory=external-sa.yaml
```

#### Auto-created service account

To have the module create a service account, set the `create` attribute to `true` and optionally pass the desired account id in `email`.

```hcl
module "cluster-1-nodepool-1" {
  source       = "./fabric/modules/gke-nodepool"
  project_id   = "myproject"
  cluster_name = "cluster-1"
  location     = "europe-west1-b"
  name         = "nodepool-1"
  service_account = {
    create       = true
    email        = "spam-eggs" # optional
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}
# tftest modules=1 resources=2 inventory=create-sa.yaml
```

### Node & node pool configuration

```hcl
module "cluster-1-nodepool-1" {
  source       = "./fabric/modules/gke-nodepool"
  project_id   = "myproject"
  cluster_name = "cluster-1"
  location     = "europe-west1-b"
  name         = "nodepool-1"
  k8s_labels   = { environment = "dev" }
  service_account = {
    create       = true
    email        = "nodepool-1" # optional
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
  node_config = {
    machine_type        = "n2-standard-2"
    disk_size_gb        = 50
    disk_type           = "pd-ssd"
    ephemeral_ssd_count = 1
    gvnic               = true
    spot                = true
  }
  nodepool_config = {
    autoscaling = {
      max_node_count = 10
      min_node_count = 1
    }
    management = {
      auto_repair  = true
      auto_upgrade = false
    }
  }
}
# tftest modules=1 resources=2 inventory=config.yaml
```

### GPU Node & node pool configuration

```hcl
module "cluster-1-nodepool-gpu-1" {
  source       = "./fabric/modules/gke-nodepool"
  project_id   = "myproject"
  cluster_name = "cluster-1"
  location     = "europe-west4-a"
  name         = "nodepool-gpu-1"
  k8s_labels   = { environment = "dev" }
  service_account = {
    create       = true
    email        = "nodepool-gpu-1" # optional
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
  node_config = {
    machine_type        = "a2-highgpu-1g"
    disk_size_gb        = 50
    disk_type           = "pd-ssd"
    ephemeral_ssd_count = 1
    gvnic               = true
    spot                = true
    guest_accelerator = {
      type  = "nvidia-tesla-a100"
      count = 1
      gpu_driver = {
        version = "LATEST"
      }
    }
  }
}
# tftest modules=1 resources=2 inventory=guest-accelerator.yaml
```

### Dynamic Workload Scheduler (DWS) & node pool configuration

This example uses Dynamic Workload Scheduler (DWS) to configure a GPU nodepool.

```hcl
module "cluster-1-nodepool-dws" {
  source       = "./fabric/modules/gke-nodepool"
  project_id   = "myproject"
  cluster_name = "cluster-1"
  location     = "europe-west4-a"
  name         = "nodepool-dws"
  k8s_labels   = { environment = "dev" }
  service_account = {
    create       = true
    email        = "nodepool-gpu-1" # optional
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
  node_config = {
    machine_type        = "g2-standard-4"
    disk_size_gb        = 50
    disk_type           = "pd-ssd"
    ephemeral_ssd_count = 1
    gvnic               = true
    spot                = true
    guest_accelerator = {
      type  = "nvidia-l4"
      count = 1
      gpu_driver = {
        version = "LATEST"
      }
    }
  }
  nodepool_config = {
    autoscaling = {
      max_node_count = 10
      min_node_count = 0
    }
    queued_provisioning = true
  }
  node_count = {
    initial = 0
  }
  reservation_affinity = {
    consume_reservation_type = "NO_RESERVATION"
  }
}
# tftest modules=1 resources=2 inventory=dws.yaml
```
<!-- BEGIN TFDOC -->
## Variables

| name | description | type | required | default |
|---|---|:---:|:---:|:---:|
| [cluster_name](variables.tf#L23) | Cluster name. | <code>string</code> | ✓ |  |
| [location](variables.tf#L48) | Cluster location. | <code>string</code> | ✓ |  |
| [project_id](variables.tf#L208) | Cluster project id. | <code>string</code> | ✓ |  |
| [cluster_id](variables.tf#L17) | Cluster id. Optional, but providing cluster_id is recommended to prevent cluster misconfiguration in some of the edge cases. | <code>string</code> |  | <code>null</code> |
| [gke_version](variables.tf#L28) | Kubernetes nodes version. Ignored if auto_upgrade is set in management_config. | <code>string</code> |  | <code>null</code> |
| [k8s_labels](variables.tf#L34) | Kubernetes labels applied to each node. | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [labels](variables.tf#L41) | The resource labels to be applied each node (vm). | <code>map&#40;string&#41;</code> |  | <code>&#123;&#125;</code> |
| [max_pods_per_node](variables.tf#L53) | Maximum number of pods per node. | <code>number</code> |  | <code>null</code> |
| [name](variables.tf#L59) | Optional nodepool name. | <code>string</code> |  | <code>null</code> |
| [network_config](variables.tf#L65) | Network configuration. | <code title="object&#40;&#123;&#10;  enable_private_nodes &#61; optional&#40;bool&#41;&#10;  pod_range &#61; optional&#40;object&#40;&#123;&#10;    cidr   &#61; optional&#40;string&#41;&#10;    create &#61; optional&#40;bool, false&#41;&#10;    name   &#61; optional&#40;string&#41;&#10;  &#125;&#41;, &#123;&#125;&#41;&#10;  additional_node_network_configs &#61; optional&#40;list&#40;object&#40;&#123;&#10;    network    &#61; string&#10;    subnetwork &#61; string&#10;  &#125;&#41;&#41;, &#91;&#93;&#41;&#10;  additional_pod_network_configs &#61; optional&#40;list&#40;object&#40;&#123;&#10;    subnetwork          &#61; string&#10;    secondary_pod_range &#61; string&#10;    max_pods_per_node   &#61; string&#10;  &#125;&#41;&#41;, &#91;&#93;&#41;&#10;  total_egress_bandwidth_tier        &#61; optional&#40;string&#41;&#10;  pod_cidr_overprovisioning_disabled &#61; optional&#40;bool, false&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [node_config](variables.tf#L89) | Node-level configuration. | <code title="object&#40;&#123;&#10;  boot_disk_kms_key   &#61; optional&#40;string&#41;&#10;  disk_size_gb        &#61; optional&#40;number&#41;&#10;  disk_type           &#61; optional&#40;string, &#34;pd-balanced&#34;&#41;&#10;  ephemeral_ssd_count &#61; optional&#40;number&#41;&#10;  gcfs                &#61; optional&#40;bool, false&#41;&#10;  guest_accelerator &#61; optional&#40;object&#40;&#123;&#10;    count &#61; number&#10;    type  &#61; string&#10;    gpu_driver &#61; optional&#40;object&#40;&#123;&#10;      version                    &#61; string&#10;      partition_size             &#61; optional&#40;string&#41;&#10;      max_shared_clients_per_gpu &#61; optional&#40;number&#41;&#10;    &#125;&#41;&#41;&#10;  &#125;&#41;&#41;&#10;  local_nvme_ssd_count &#61; optional&#40;number&#41;&#10;  gvnic                &#61; optional&#40;bool, false&#41;&#10;  image_type           &#61; optional&#40;string&#41;&#10;  kubelet_config &#61; optional&#40;object&#40;&#123;&#10;    cpu_manager_policy   &#61; string&#10;    cpu_cfs_quota        &#61; optional&#40;bool&#41;&#10;    cpu_cfs_quota_period &#61; optional&#40;string&#41;&#10;    pod_pids_limit       &#61; optional&#40;number&#41;&#10;  &#125;&#41;&#41;&#10;  linux_node_config &#61; optional&#40;object&#40;&#123;&#10;    sysctls     &#61; optional&#40;map&#40;string&#41;&#41;&#10;    cgroup_mode &#61; optional&#40;string&#41;&#10;  &#125;&#41;&#41;&#10;  local_ssd_count       &#61; optional&#40;number&#41;&#10;  machine_type          &#61; optional&#40;string&#41;&#10;  metadata              &#61; optional&#40;map&#40;string&#41;&#41;&#10;  min_cpu_platform      &#61; optional&#40;string&#41;&#10;  preemptible           &#61; optional&#40;bool&#41;&#10;  sandbox_config_gvisor &#61; optional&#40;bool&#41;&#10;  shielded_instance_config &#61; optional&#40;object&#40;&#123;&#10;    enable_integrity_monitoring &#61; optional&#40;bool&#41;&#10;    enable_secure_boot          &#61; optional&#40;bool&#41;&#10;  &#125;&#41;&#41;&#10;  spot                          &#61; optional&#40;bool&#41;&#10;  workload_metadata_config_mode &#61; optional&#40;string&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>&#123;&#125;</code> |
| [node_count](variables.tf#L154) | Number of nodes per instance group. Initial value can only be changed by recreation, current is ignored when autoscaling is used. | <code title="object&#40;&#123;&#10;  current &#61; optional&#40;number&#41;&#10;  initial &#61; number&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code title="&#123;&#10;  initial &#61; 1&#10;&#125;">&#123;&#8230;&#125;</code> |
| [node_locations](variables.tf#L166) | Node locations. | <code>list&#40;string&#41;</code> |  | <code>null</code> |
| [nodepool_config](variables.tf#L172) | Nodepool-level configuration. | <code title="object&#40;&#123;&#10;  autoscaling &#61; optional&#40;object&#40;&#123;&#10;    location_policy &#61; optional&#40;string&#41;&#10;    max_node_count  &#61; optional&#40;number&#41;&#10;    min_node_count  &#61; optional&#40;number&#41;&#10;    use_total_nodes &#61; optional&#40;bool, false&#41;&#10;  &#125;&#41;&#41;&#10;  management &#61; optional&#40;object&#40;&#123;&#10;    auto_repair  &#61; optional&#40;bool&#41;&#10;    auto_upgrade &#61; optional&#40;bool&#41;&#10;  &#125;&#41;&#41;&#10;  placement_policy &#61; optional&#40;object&#40;&#123;&#10;    type         &#61; string&#10;    policy_name  &#61; optional&#40;string&#41;&#10;    tpu_topology &#61; optional&#40;string&#41;&#10;  &#125;&#41;&#41;&#10;  queued_provisioning &#61; optional&#40;bool, false&#41;&#10;  upgrade_settings &#61; optional&#40;object&#40;&#123;&#10;    max_surge       &#61; number&#10;    max_unavailable &#61; number&#10;    strategy        &#61; optional&#40;string&#41;&#10;    blue_green_settings &#61; optional&#40;object&#40;&#123;&#10;      node_pool_soak_duration &#61; optional&#40;string&#41;&#10;      standard_rollout_policy &#61; optional&#40;object&#40;&#123;&#10;        batch_percentage    &#61; optional&#40;number&#41;&#10;        batch_node_count    &#61; optional&#40;number&#41;&#10;        batch_soak_duration &#61; optional&#40;string&#41;&#10;      &#125;&#41;&#41;&#10;    &#125;&#41;&#41;&#10;  &#125;&#41;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [reservation_affinity](variables.tf#L213) | Configuration of the desired reservation which instances could take capacity from. | <code title="object&#40;&#123;&#10;  consume_reservation_type &#61; string&#10;  key                      &#61; optional&#40;string&#41;&#10;  values                   &#61; optional&#40;list&#40;string&#41;&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>null</code> |
| [service_account](variables.tf#L223) | Nodepool service account. If this variable is set to null, the default GCE service account will be used. If set and email is null, a service account will be created. If scopes are null a default will be used. | <code title="object&#40;&#123;&#10;  create       &#61; optional&#40;bool, false&#41;&#10;  email        &#61; optional&#40;string&#41;&#10;  oauth_scopes &#61; optional&#40;list&#40;string&#41;&#41;&#10;  display_name &#61; optional&#40;string&#41;&#10;&#125;&#41;">object&#40;&#123;&#8230;&#125;&#41;</code> |  | <code>&#123;&#125;</code> |
| [sole_tenant_nodegroup](variables.tf#L235) | Sole tenant node group. | <code>string</code> |  | <code>null</code> |
| [tags](variables.tf#L241) | Network tags applied to nodes. | <code>list&#40;string&#41;</code> |  | <code>null</code> |
| [taints](variables.tf#L247) | Kubernetes taints applied to all nodes. | <code title="map&#40;object&#40;&#123;&#10;  value  &#61; string&#10;  effect &#61; string&#10;&#125;&#41;&#41;">map&#40;object&#40;&#123;&#8230;&#125;&#41;&#41;</code> |  | <code>&#123;&#125;</code> |

## Outputs

| name | description | sensitive |
|---|---|:---:|
| [id](outputs.tf#L17) | Fully qualified nodepool id. |  |
| [name](outputs.tf#L22) | Nodepool name. |  |
| [service_account_email](outputs.tf#L27) | Service account email. |  |
| [service_account_iam_email](outputs.tf#L32) | Service account email. |  |
<!-- END TFDOC -->

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| cluster\_id | Cluster id. Optional, but providing cluster\_id is recommended to prevent cluster misconfiguration in some of the edge cases. | `string` | `null` | no |
| cluster\_name | Cluster name. | `string` | n/a | yes |
| gke\_version | Kubernetes nodes version. Ignored if auto\_upgrade is set in management\_config. | `string` | `null` | no |
| k8s\_labels | Kubernetes labels applied to each node. | `map(string)` | `{}` | no |
| labels | The resource labels to be applied each node (vm). | `map(string)` | `{}` | no |
| location | Cluster location. | `string` | n/a | yes |
| max\_pods\_per\_node | Maximum number of pods per node. | `number` | `null` | no |
| name | Optional nodepool name. | `string` | `null` | no |
| network\_config | Network configuration. | <pre>object({<br>    enable_private_nodes = optional(bool, true)<br>    pod_range = optional(object({<br>      cidr   = optional(string)<br>      create = optional(bool, false)<br>      name   = optional(string)<br>    }), {})<br>    additional_node_network_configs = optional(list(object({<br>      network    = string<br>      subnetwork = string<br>    })), [])<br>    additional_pod_network_configs = optional(list(object({<br>      subnetwork          = string<br>      secondary_pod_range = string<br>      max_pods_per_node   = string<br>    })), [])<br>    total_egress_bandwidth_tier        = optional(string)<br>    pod_cidr_overprovisioning_disabled = optional(bool, false)<br>  })</pre> | <pre>{<br>  "enable_private_nodes": true<br>}</pre> | no |
| node\_config | Node-level configuration. | <pre>object({<br>    boot_disk_kms_key   = optional(string)<br>    disk_size_gb        = optional(number)<br>    disk_type           = optional(string, "pd-balanced")<br>    ephemeral_ssd_count = optional(number)<br>    gcfs                = optional(bool, false)<br>    guest_accelerator = optional(object({<br>      count = number<br>      type  = string<br>      gpu_driver = optional(object({<br>        version                    = string<br>        partition_size             = optional(string)<br>        max_shared_clients_per_gpu = optional(number)<br>      }))<br>    }))<br>    local_nvme_ssd_count = optional(number)<br>    gvnic                = optional(bool, false)<br>    image_type           = optional(string)<br>    kubelet_config = optional(object({<br>      cpu_manager_policy   = string<br>      cpu_cfs_quota        = optional(bool)<br>      cpu_cfs_quota_period = optional(string)<br>      pod_pids_limit       = optional(number)<br>    }))<br>    linux_node_config = optional(object({<br>      sysctls     = optional(map(string))<br>      cgroup_mode = optional(string)<br>    }))<br>    local_ssd_count       = optional(number)<br>    machine_type          = optional(string)<br>    metadata              = optional(map(string))<br>    min_cpu_platform      = optional(string)<br>    preemptible           = optional(bool)<br>    sandbox_config_gvisor = optional(bool)<br>    shielded_instance_config = optional(object({<br>      enable_integrity_monitoring = optional(bool)<br>      enable_secure_boot          = optional(bool)<br>    }))<br>    spot                          = optional(bool)<br>    workload_metadata_config_mode = optional(string)<br>  })</pre> | `{}` | no |
| node\_count | Number of nodes per instance group. Initial value can only be changed by recreation, current is ignored when autoscaling is used. | <pre>object({<br>    current = optional(number)<br>    initial = number<br>  })</pre> | <pre>{<br>  "initial": 1<br>}</pre> | no |
| node\_locations | Node locations. | `list(string)` | `null` | no |
| nodepool\_config | Nodepool-level configuration. | <pre>object({<br>    autoscaling = optional(object({<br>      location_policy = optional(string)<br>      max_node_count  = optional(number)<br>      min_node_count  = optional(number)<br>      use_total_nodes = optional(bool, false)<br>    }))<br>    management = optional(object({<br>      auto_repair  = optional(bool)<br>      auto_upgrade = optional(bool)<br>    }))<br>    placement_policy = optional(object({<br>      type         = string<br>      policy_name  = optional(string)<br>      tpu_topology = optional(string)<br>    }))<br>    queued_provisioning = optional(bool, false)<br>    upgrade_settings = optional(object({<br>      max_surge       = number<br>      max_unavailable = number<br>      strategy        = optional(string)<br>      blue_green_settings = optional(object({<br>        node_pool_soak_duration = optional(string)<br>        standard_rollout_policy = optional(object({<br>          batch_percentage    = optional(number)<br>          batch_node_count    = optional(number)<br>          batch_soak_duration = optional(string)<br>        }))<br>      }))<br>    }))<br>  })</pre> | `null` | no |
| project\_id | Cluster project id. | `string` | n/a | yes |
| reservation\_affinity | Configuration of the desired reservation which instances could take capacity from. | <pre>object({<br>    consume_reservation_type = string<br>    key                      = optional(string)<br>    values                   = optional(list(string))<br>  })</pre> | `null` | no |
| service\_account | Nodepool service account. If this variable is set to null, the default GCE service account will be used. If set and email is null, a service account will be created. If scopes are null a default will be used. | <pre>object({<br>    create       = optional(bool, false)<br>    email        = optional(string)<br>    oauth_scopes = optional(list(string))<br>    display_name = optional(string)<br>  })</pre> | <pre>{<br>  "create": false<br>}</pre> | no |
| sole\_tenant\_nodegroup | Sole tenant node group. | `string` | `null` | no |
| tags | Network tags applied to nodes. | `list(string)` | `null` | no |
| taints | Kubernetes taints applied to all nodes. | <pre>map(object({<br>    value  = string<br>    effect = string<br>  }))</pre> | `{}` | no |

## Outputs

| Name | Description |
|------|-------------|
| id | Fully qualified nodepool id. |
| name | Nodepool name. |
| service\_account\_email | Service account email. |
| service\_account\_iam\_email | Service account email. |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->