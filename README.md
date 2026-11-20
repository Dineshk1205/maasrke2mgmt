# RKE2 + Rancher on MAAS

Cloud-init files and Rancher UI values to build two RKE2 clusters on machines deployed by MAAS:

- **rke-mgmt**: 3-node management cluster with Rancher, Longhorn and monitoring (Prometheus + Grafana).
- **ai-prod**: 5-node workload cluster (3 control plane + 2 GPU workers), created from Rancher.
  Add-ons and AI apps are then installed by Fleet from a separate GitOps repo.

MAAS installs Ubuntu and runs the cloud-init file you paste in the deploy screen. Everything else
(packages, RKE2, Rancher, registration) happens automatically from these files. No manual steps on
the nodes are needed, except creating one secret (see step 5).

The files work the same on bare metal and on VMs (vSphere, Proxmox, KVM). Only the GPU driver
part depends on how the GPU is attached (see [GPU nodes](#gpu-nodes)).

## Repository layout

```
.
├── README.md
├── cloud-init/              paste into MAAS > Deploy > Cloud-init user-data
│   ├── haproxy-user-data.yaml
│   ├── mgmt01-user-data.yaml
│   ├── mgmt02-user-data.yaml
│   ├── mgmt03-user-data.yaml
│   ├── wrkldc-cp-user-data.yaml
│   └── wrkldc-gpu-user-data.yaml
└── rancher-ui-values/       values for apps installed from the Rancher UI
    ├── monitoring-dashboards-values.yaml
    └── neuvector-values.yaml
```

### cloud-init/

| File | Used for | Role |
|---|---|---|
| `haproxy-user-data.yaml` | haproxy | Load balancer with two VIPs (mgmt and ai-prod) |
| `mgmt01-user-data.yaml` | mgmt01 | First RKE2 server; also installs cert-manager, Rancher, Longhorn, kube-prometheus-stack and the remote-write ingress |
| `mgmt02-user-data.yaml` | mgmt02 | Joins the mgmt cluster |
| `mgmt03-user-data.yaml` | mgmt03 | Joins the mgmt cluster |
| `wrkldc-cp-user-data.yaml` | wrkldc01-03 | Registers with Rancher as etcd + control plane |
| `wrkldc-gpu-user-data.yaml` | wrkldc04-05 | Installs NVIDIA vGPU guest driver, registers with Rancher as worker (`gpu=true`) |

### rancher-ui-values/

Values for apps installed from the Rancher UI (**Cluster Tools** or **Apps > Charts**).
Open the app, click **Install**, choose **Edit YAML**, and set only the keys in the file.

| File | Cluster | App |
|---|---|---|
| `monitoring-dashboards-values.yaml` | local (mgmt) | Monitoring Dashboards (uses the kube-prometheus-stack installed by mgmt01) |
| `neuvector-values.yaml` | local (mgmt) | NeuVector |

Every line you may need to change is marked with `# CHANGE:` in the files.
Search for it: `grep -rn "CHANGE" .`

## Versions

| Component | Version |
|---|---|
| Ubuntu | LTS release from MAAS (same on all nodes) |
| RKE2 | v1.35.10+rke2r1 |
| Rancher | 2.15.3 |
| cert-manager | v1.21.2 |
| Longhorn | 1.13.0 |
| kube-prometheus-stack | 93.4.0 |
| NVIDIA vGPU guest driver (GRID) | 595.112.07 |

## Example network plan

All files use this plan. Change the IPs and names to match your network.

| Name | IP | Notes |
|---|---|---|
| Gateway | 172.16.40.1 | VLAN 40, 172.16.40.0/24 |
| DNS server | 172.16.10.5 | |
| maas | 172.16.40.3 | Also serves the GPU driver files on port 8081 |
| haproxy | 172.16.40.4 | |
| mgmt01 / mgmt02 / mgmt03 | 172.16.40.5 / .6 / .7 | |
| wrkldc01 / 02 / 03 | 172.16.40.8 / .9 / .10 | ai-prod control plane |
| wrkldc04 / 05 | 172.16.40.11 / .12 | ai-prod GPU workers |
| VIP-MGMT | 172.16.40.20 | On haproxy |
| VIP-WKLD | 172.16.40.21 | On haproxy |
| MetalLB pool (ai-prod) | 172.16.40.50-.69 | Used by the GitOps repo |
| MAAS dynamic range | 172.16.40.200-.229 | PXE / commissioning only |

DNS records (domain `gpu.lab`):

| Record | Points to |
|---|---|
| `rke-mgmt.gpu.lab`, `rancher.gpu.lab`, `prom-write.gpu.lab` | 172.16.40.20 (VIP-MGMT) |
| `ai-prod.gpu.lab`, `ai.gpu.lab`, `*.apps.gpu.lab` | 172.16.40.21 (VIP-WKLD) |
| `mgmt01..03.gpu.lab`, `wrkldc01..05.gpu.lab` | node IPs |

## What you must change

### 1. Placeholders (secrets, never commit real values)

| Placeholder | Files | How to get it |
|---|---|---|
| `<RKE2_CLUSTER_TOKEN>` | mgmt01, mgmt02, mgmt03 | Any random string, same in all three. Example: `openssl rand -hex 16` |
| `<RANCHER_BOOTSTRAP_PASSWORD>` | mgmt01 | Password for the first Rancher login (user `admin`) |
| `<RANCHER_REGISTRATION_TOKEN>` | wrkldc-cp, wrkldc-gpu | Rancher > Cluster Management > your cluster > Registration: value after `--token` |
| `<RANCHER_CA_CHECKSUM>` | wrkldc-cp, wrkldc-gpu | Same registration command: value after `--ca-checksum` |
| `<NEUVECTOR_ADMIN_PASSWORD>` | rancher-ui-values/neuvector-values.yaml | Password for the NeuVector `admin` login |

Make a local copy with real values and use that. Keep the copy out of Git.

### 2. Lab values (change to your environment)

| Value in files | Meaning | Files |
|---|---|---|
| `172.16.40.x` | Node IPs and VIPs | haproxy, mgmt01-03 |
| `gpu.lab` | DNS domain (`rke-mgmt`, `rancher`, `prom-write`, node names) | mgmt01-03, wrkldc-cp, wrkldc-gpu |
| `/var/lib/longhorn/` | Mount point of the Longhorn data disk | mgmt01 |
| `defaultClassReplicaCount: 3` | Longhorn replicas, not more than the number of nodes | mgmt01 |
| `retention: 1d`, `8GiB`, `10Gi` | Prometheus retention and disk size (lab sizes) | mgmt01 |
| `http://172.16.40.3:8081/...` | Internal web server with the GRID driver and licence token | wrkldc-gpu |
| `nvidia-linux-grid-595_595.112.07_amd64.deb` | GRID guest driver file name | wrkldc-gpu |
| `INSTALL_RKE2_VERSION`, chart `version:` | Versions, keep the same on all mgmt nodes | mgmt01-03 |

## MAAS settings per machine

For every machine, before Deploy:

1. **Network**: set the interface to **Static assign** with the IP from the plan.
   For haproxy, also add the two VIPs (172.16.40.20 and .21) as extra IPs on the same interface.
2. **Storage**: OS on the first disk. On mgmt and wrkldc nodes, format the second disk (`sdb`) as
   ext4 and mount it at `/var/lib/longhorn`.
3. **Deploy**: Ubuntu LTS, open **Cloud-init user-data**, paste the matching file.

Power control:
- **Bare metal**: MAAS powers machines on and off through the BMC (IPMI/Redfish). Fully automatic.
- **VMs**: with power type **Manual**, power the VM on yourself after Commission / Deploy, and off
  after Release. MAAS also has VMware, Proxmox and virsh power types if you want MAAS to do it.

## Deployment order

### 1. HAProxy

Deploy `haproxy` with `haproxy-user-data.yaml`.
Check: `http://172.16.40.4:8404/` shows the stats page (backends are DOWN until nodes exist).

### 2. Management cluster

1. Deploy **mgmt01** with `mgmt01-user-data.yaml`. Wait until it is Ready:
   ```bash
   sudo -i
   kubectl get nodes
   ```
2. Deploy **mgmt02** and **mgmt03** (both at the same time is fine).
3. Watch the add-ons install (takes 5-15 minutes):
   ```bash
   kubectl get helmcharts -n kube-system
   kubectl get pods -A
   ```
   The first Rancher install job may fail once while cert-manager is starting. It retries by itself.

### 3. Rancher

Open `https://rancher.gpu.lab` and log in as `admin` with `<RANCHER_BOOTSTRAP_PASSWORD>`.
Set a new password and confirm the server URL.

### 4. Cluster tools on mgmt (Rancher UI)

In **local > Cluster Tools**, install with the values from `rancher-ui-values/`:

1. **Monitoring Dashboards**: `monitoring-dashboards-values.yaml`
2. **NeuVector**: `neuvector-values.yaml`
3. **Compliance**: defaults, then run a scan with profile `rke2-cis-2.0-profile`

### 5. Remote-write secret (one time, on mgmt01)

Workload clusters send metrics to `https://prom-write.gpu.lab/api/v1/write` with basic auth.
Create the password file and secret (choose your own user and password):

```bash
apt-get install -y apache2-utils
htpasswd -c auth promwriter
kubectl -n cattle-monitoring-system create secret generic prom-remote-write-auth --from-file=auth
```

The same user and password go into the `prom-remote-write` secret on ai-prod
(see the GitOps repo README).

### 6. Workload cluster (ai-prod)

1. In Rancher: **Cluster Management > Create > Custom**:
   - Name: `ai-prod`
   - Kubernetes version: `v1.35.10+rke2r1`
   - CNI: **Cilium**
   - CIS profile on, Pod Security Admission template `rancher-restricted`
   - Add a label `env=ai-prod` (Fleet uses it to target the cluster)
2. Open the **Registration** tab and copy the `--token` and `--ca-checksum` values into
   your local copies of `wrkldc-cp-user-data.yaml` and `wrkldc-gpu-user-data.yaml`.
3. Deploy **wrkldc01-03** with `wrkldc-cp-user-data.yaml`.
   They appear in Rancher under Machines and Rancher installs RKE2 on them.
4. Deploy **wrkldc04-05** with `wrkldc-gpu-user-data.yaml`.
   They install the driver, reboot once, check `nvidia-smi`, then join as workers.
5. In Rancher > Continuous Delivery, add the GitOps repo for add-ons and AI apps.


## GPU nodes

`wrkldc-gpu-user-data.yaml` is written for **NVIDIA vGPU** (a vGPU profile on a VM, e.g.
`nvidia_rtxa6000-12q` on vSphere). It needs:

- The **GRID guest driver** `.deb`, matching the vGPU Manager version on the host.
- The **licence token** `client_configuration_token.tok`.

Both are NVIDIA licensed files. Do **not** put them in Git. Serve them from an internal web server
the nodes can reach, for example on the MAAS server:

```bash
mkdir -p /srv/nvidia && cd /srv/nvidia   # copy the .deb and .tok here
python3 -m http.server 8081
```

For **GPU passthrough** or **bare-metal GPUs**: remove the lines between `VGPU-ONLY start` and
`VGPU-ONLY end` and the `ExecStartPre=/usr/bin/nvidia-smi` line, and set `driver.enabled=true`
in the GPU Operator values so the operator installs the driver.

## Check the result

```bash
# mgmt cluster (on mgmt01)
kubectl get nodes -o wide
kubectl get pods -A
kubectl -n cattle-system get ingress

# a wrkldc node, if registration seems stuck
systemctl status rancher-register.service
journalctl -u rancher-register.service -f

# GPU node
nvidia-smi
nvidia-smi -q | grep -i license
```

##  notes

- `--insecure` in the registration command is used because Rancher runs with its own self-signed CA.
  `--ca-checksum` still verifies the CA.
- The registration token lets a machine join the cluster. Treat your filled-in copies as secrets.
- RKE2 runs with the CIS profile and Pod Security Admission `restricted` by default.
