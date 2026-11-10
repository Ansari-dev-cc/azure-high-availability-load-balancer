# Cloud High Availability: Load Balancer and Availability Set Integration

Designing, deploying, monitoring, and failover-testing a fault-tolerant, highly available web application architecture on **Microsoft Azure**.

## 1. Project Overview & Architecture

This project implements a horizontally redundant cloud web infrastructure on **Microsoft Azure** designed to eliminate single points of failure (SPOFs). By deploying two identical **Ubuntu Server 24.04 LTS** NGINX web servers (`vm-web1` and `vm-web2`) inside an **Azure Availability Set (`avset-web`)** configured with **2 Fault Domains** and **5 Update Domains**, and fronting them with an **Azure Standard Public Load Balancer (`lb-web`)** at `4.246.239.103`, incoming client requests are automatically distributed across healthy backend nodes. Continuous TCP Port 80 health probes detect any node outage within seconds and seamlessly reroute traffic to the remaining active instance with zero application downtime.

```text
                        +---------------------------------------+
                        |      Internet / Client Browsers       |
                        +-------------------+-------------------+
                                            |
                                  HTTP (TCP Port 80)
                                            |
                                            v
                        +---------------------------------------+
                        |   Azure Standard Load Balancer        |
                        |   Name: lb-web                        |
                        |   Frontend IP: 4.246.239.103 (pip-lb) |
                        |   Health Probe: probe-http (TCP:80)   |
                        |   LB Rule: rule-http (TCP 80 -> 80)   |
                        +---------+-------------------+---------+
                                  |                   |
             +--------------------+                   +--------------------+
             | 5-Tuple Hash / Probe                   | 5-Tuple Hash / Probe
             v                                        v
+----------------------------------+    +----------------------------------+
| Virtual Network: vnet-ha (10.0.0.0/16) / Subnet: subnet-web (10.0.1.0/24)|
| Network Security Group: nsg-web (Allow-HTTP: 80 [100], Allow-SSH: 22)    |
|                                                                          |
|  +-------- Availability Set: avset-web (2 Fault / 5 Update Domains) --+  |
|  |                                                                    |  |
|  |  +-----------------------------+  +-----------------------------+  |  |
|  |  | Node 1: vm-web1 (FD0 / UD0) |  | Node 2: vm-web2 (FD1 / UD1) |  |  |
|  |  | Size: Standard D2als v7     |  | Size: Standard D2als v7     |  |  |
|  |  | OS: Ubuntu 24.04 LTS + NGINX|  | OS: Ubuntu 24.04 LTS + NGINX|  |  |
|  |  | Private IP: 10.0.1.4        |  | Private IP: 10.0.1.5        |  |  |
|  |  | Public IP: 13.82.138.244    |  | Public IP: 4.227.208.195    |  |  |
|  |  +-----------------------------+  +-----------------------------+  |  |
|  +--------------------------------------------------------------------+  |
+--------------------------------------------------------------------------+
```

## 2. Repository Directory Structure

```text
.
├── README.md                                                     # Project documentation
├── .gitignore                                                    # Git ignore rules
├── docs/
│   └── Azure_High_Availability_Load_Balancer_Report.pdf          # 16-Page Technical Report (PDF)
├── scripts/
│   ├── deploy_azure_ha.sh                                        # End-to-end Azure CLI provisioning script
│   ├── setup_nginx_node1.sh                                      # NGINX & HTML setup script for vm-web1
│   ├── setup_nginx_node2.sh                                      # NGINX & HTML setup script for vm-web2
│   └── stress_test_vm.sh                                         # CPU stress test & cleanup script
├── src/
│   ├── vm-web1/index.html                                        # Node 1 HTML payload
│   └── vm-web2/index.html                                        # Node 2 HTML payload
└── screenshots/                                                  # 45 step-by-step verification screenshots
```

## 3. Cloud Resources Inventory

| Resource Name | Azure Resource Type | SKU / CIDR / Configuration | Role in High Availability Architecture |
| :--- | :--- | :--- | :--- |
| `RG-CoreTask4` | Resource Group | Region: `East US` (`eastus`) | Logical container for all cloud infrastructure assets |
| `vnet-ha` | Virtual Network | Address Space: `10.0.0.0/16` | Isolated software-defined network in East US |
| `subnet-web` | Subnet | Address Range: `10.0.1.0/24` (256 IPs) | Dedicated web tier subnet hosting backend VMs |
| `nsg-web` | Network Security Group | `Allow-HTTP` (TCP/80, Pri 100), `Allow-SSH` (TCP/22, Pri 110) | Subnet-level firewall attached to `subnet-web` |
| `avset-web` | Availability Set | 2 Fault Domains, 5 Update Domains | Hardware rack & rolling host update isolation |
| `vm-web1` | Virtual Machine | `Standard D2als v7` (2 vCPUs, 4 GiB RAM), Ubuntu 24.04 | Primary web server (`10.0.1.4` / `13.82.138.244`) |
| `vm-web2` | Virtual Machine | `Standard D2als v7` (2 vCPUs, 4 GiB RAM), Ubuntu 24.04 | Secondary web server (`10.0.1.5` / `4.227.208.195`) |
| `lb-web` | Load Balancer | Standard SKU, Public, Regional | Layer-4 TCP traffic distributor across backend VMs |
| `pip-lb` | Public IP Address | `4.246.239.103` (Standard, Static, Zone-Redundant) | Frontend public IP (`lb-frontend`) for client traffic |
| `backend-pool` | Backend Pool | NIC-based (`10.0.1.4`, `10.0.1.5`) | Pool of active Availability Set VM interfaces |
| `probe-http` | Health Probe | Protocol: `TCP`, Port: `80`, Interval: `5s` | Tracks live NGINX socket status on backend nodes |
| `rule-http` | Load Balancing Rule | TCP `80 -> 80`, Session Persistence: `None`, Idle: `4m` | Distributes HTTP requests across healthy VMs |

## 4. Step-by-Step Implementation & Evidence

### Phase 1: Resource Group, Virtual Network (`vnet-ha`) & Subnet (`subnet-web`)
1. Created Resource Group `RG-CoreTask4` in `(US) East US`.
2. Provisioned Virtual Network `vnet-ha` (`10.0.0.0/16`) and added subnet `subnet-web` (`10.0.1.0/24`).

| Create Resource Group | Configure `subnet-web` (`10.0.1.0/24`) |
| :---: | :---: |
| ![Create RG](./screenshots/01-create-resource-group.png) | ![Add Subnet](./screenshots/02-add-subnet-web-10-0-1-0-24.png) |
| **Address Space & Subnets Overview** | **Virtual Network `vnet-ha` Basics** |
| ![VNet Subnets](./screenshots/03-vnet-address-space-and-subnets.png) | ![VNet Basics](./screenshots/04-create-vnet-ha-basics.png) |

### Phase 2: Network Security Group (`nsg-web`) & Subnet Association
1. Created Network Security Group `nsg-web`.
2. Added inbound rules `Allow-HTTP` (TCP Port 80, Priority 100) and `Allow-SSH` (TCP Port 22, Priority 110).
3. Associated `nsg-web` directly with `subnet-web` (`10.0.1.0/24`) inside `vnet-ha`.

| Create `nsg-web` | Default Inbound Rules |
| :---: | :---: |
| ![Create NSG](./screenshots/05-create-nsg-web.png) | ![Default Rules](./screenshots/06-nsg-web-default-inbound-rules.png) |
| **Add `Allow-HTTP` (Port 80, Priority 100)** | **Add `Allow-SSH` (Port 22, Priority 110)** |
| ![Allow HTTP](./screenshots/07-nsg-add-allow-http-port-80.png) | ![Allow SSH](./screenshots/08-nsg-add-allow-ssh-port-22.png) |
| **Associate `nsg-web` with `subnet-web`** | **Verified `subnet-web` Association** |
| ![Associate Subnet](./screenshots/09-nsg-associate-subnet-web.png) | ![Subnet Associated](./screenshots/10-nsg-subnet-web-associated.png) |

### Phase 3: Availability Set (`avset-web`) & Ubuntu 24.04 VMs (`vm-web1`, `vm-web2`)
1. Created Availability Set `avset-web` with **2 Fault Domains** and **5 Update Domains**.
2. Deployed `vm-web1` (`10.0.1.4` / `13.82.138.244`) and `vm-web2` (`10.0.1.5` / `4.227.208.195`) inside `avset-web` using `Ubuntu Server 24.04 LTS - x64 Gen2` (`Standard D2als v7`).

| Create Availability Set `avset-web` | Create `vm-web1` in `avset-web` |
| :---: | :---: |
| ![Create AvSet](./screenshots/11-create-availability-set-avset-web.png) | ![Create vm-web1](./screenshots/12-create-vm-web1-basics-avset.png) |
| **`vm-web1` Administrator Account (`Azureuser`)** | **`vm-web1` Networking (`subnet-web`)** |
| ![vm-web1 Auth](./screenshots/13-create-vm-web1-auth-ssh.png) | ![vm-web1 Net](./screenshots/14-create-vm-web1-networking.png) |
| **Create `vm-web2` in `avset-web`** | **`vm-web2` Networking (`subnet-web`)** |
| ![Create vm-web2](./screenshots/15-create-vm-web2-basics-avset.png) | ![vm-web2 Net](./screenshots/16-create-vm-web2-networking.png) |

### Phase 4: Web Server Installation (NGINX) & Direct Node Verification
Connected to both VMs via SSH and installed NGINX with custom node-identification HTML pages:

```bash
# On vm-web1 (13.82.138.244 / 10.0.1.4)
sudo apt update && sudo apt install nginx -y
echo "<h1>Hello from Web Server 1 (vm-web1)</h1><p>Highly Available Web App</p>" | sudo tee /var/www/html/index.html
sudo systemctl enable nginx && sudo systemctl status nginx

# On vm-web2 (4.227.208.195 / 10.0.1.5)
sudo apt update && sudo apt install nginx -y
echo "<h1>Hello from Web Server 2 (vm-web2)</h1><p>Highly Available Web App</p>" | sudo tee /var/www/html/index.html
sudo systemctl enable nginx && sudo systemctl status nginx
```

| SSH Login to `vm-web1` (`13.82.138.244`) | Install NGINX on `vm-web1` |
| :---: | :---: |
| ![SSH vm-web1](./screenshots/17-ssh-connect-vm-web1.png) | ![Install NGINX 1](./screenshots/18-vm-web1-apt-install-nginx.png) |
| **Verify `nginx.service` on `vm-web1`** | **SSH Login to `vm-web2` (`4.227.208.195`)** |
| ![Status NGINX 1](./screenshots/19-vm-web1-nginx-systemctl-status.png) | ![SSH vm-web2](./screenshots/20-ssh-connect-vm-web2.png) |
| **Install NGINX on `vm-web2`** | **Verify `nginx.service` on `vm-web2`** |
| ![Install NGINX 2](./screenshots/21-vm-web2-apt-install-nginx.png) | ![Status NGINX 2](./screenshots/22-vm-web2-nginx-systemctl-status.png) |
| **Direct HTTP Verification (`http://13.82.138.244`)** | **Direct HTTP Verification (`http://4.227.208.195`)** |
| ![Direct Test 1](./screenshots/23-browser-direct-test-vm-web1.png) | ![Direct Test 2](./screenshots/24-browser-direct-test-vm-web2.png) |

### Phase 5: Azure Standard Load Balancer (`lb-web`), Backend Pool, Health Probe & Rules
1. Created Standard Public Load Balancer `lb-web` with zone-redundant static Frontend IP `pip-lb` (`4.246.239.103`).
2. Registered `vm-web1` (`10.0.1.4`) and `vm-web2` (`10.0.1.5`) in `backend-pool`.
3. Configured TCP Port 80 Health Probe `probe-http` and Load-Balancing Rule `rule-http`.

| Create Load Balancer `lb-web` | Add Frontend IP `lb-frontend` (`pip-lb`) |
| :---: | :---: |
| ![Create LB](./screenshots/25-create-load-balancer-lb-web.png) | ![Add Frontend IP](./screenshots/26-lb-add-frontend-ip-pip-lb.png) |
| **Frontend IP Summary** | **Register VMs in `backend-pool`** |
| ![Frontend Summary](./screenshots/27-lb-frontend-ip-configured.png) | ![Add Backend Pool](./screenshots/28-lb-add-backend-pool-vms.png) |
| **Confirmed `backend-pool` Members** | **Configure `rule-http` & `probe-http`** |
| ![Backend Pool Summary](./screenshots/29-lb-backend-pool-summary.png) | ![Add Rule HTTP](./screenshots/30-lb-add-rule-http-and-probe.png) |
| **Inbound Rules Summary** | **Deployed `lb-web` Overview (`4.246.239.103`)** |
| ![Inbound Rules](./screenshots/31-lb-inbound-rules-summary.png) | ![LB Overview](./screenshots/32-lb-web-overview-deployed.png) |

### Phase 6: Traffic Distribution, Failover Simulation & Resource Monitoring
1. **Traffic Distribution**: Verified requests to `http://4.246.239.103` routed across both `vm-web1` and `vm-web2`.
2. **Failure Scenario Simulation**: Stopped `vm-web1` in the Azure Portal; verified uninterrupted application availability via automatic failover to `vm-web2`.
3. **CPU Stress Test & Azure Monitor Telemetry**: Ran `stress --cpu 2 --timeout 300` on `vm-web1` and monitored VM Percentage CPU and Load Balancer Health Probe Status (`23.26k` Sum / `250` Count).

| Load Balancer Serving `vm-web2` | Load Balancer Serving `vm-web1` |
| :---: | :---: |
| ![LB Serving Node 2](./screenshots/34-lb-frontend-ip-serving-vm-web2.png) | ![LB Serving Node 1](./screenshots/41-lb-frontend-ip-serving-vm-web1.png) |
| **Simulating Outage: Stopping `vm-web1`** | **Automatic Failover Verified on `vm-web2`** |
| ![Stop vm-web1](./screenshots/42-simulate-failure-stop-vm-web1.png) | ![Failover Verified](./screenshots/43-failover-verified-on-vm-web2.png) |
| **Running `stress` CPU Load on `vm-web1`** | **Removing `stress` After Telemetry Capture** |
| ![Run Stress](./screenshots/33-vm-web1-install-run-stress.png) | ![Remove Stress](./screenshots/40-vm-web1-stop-and-remove-stress.png) |
| **Azure Monitor: `vm-web1` Percentage CPU Spike** | **Azure Monitor: `vm-web1` CPU Time-Series** |
| ![CPU Spike](./screenshots/37-azure-monitor-vm-web1-cpu-spike.png) | ![CPU Tooltip](./screenshots/38-azure-monitor-vm-web1-cpu-tooltip.png) |
| **Azure Monitor: `lb-web` Health Probe Status (Sum)** | **Azure Monitor: `lb-web` Health Probe Status (Count)** |
| ![Probe Sum](./screenshots/44-azure-monitor-lb-health-probe-sum.png) | ![Probe Count](./screenshots/45-azure-monitor-lb-health-probe-count.png) |

## 5. Verification Results Summary

| Verification Scenario | Target Endpoint / Resource | Observed Outcome | Status |
| :--- | :--- | :--- | :--- |
| **Direct Node 1 HTTP Test** | `http://13.82.138.244` (`10.0.1.4`) | Rendered `Hello from Web Server 1 (vm-web1)` | PASS |
| **Direct Node 2 HTTP Test** | `http://4.227.208.195` (`10.0.1.5`) | Rendered `Hello from Web Server 2 (vm-web2)` | PASS |
| **Load-Balanced Distribution** | `http://4.246.239.103` (`pip-lb`) | Distributed HTTP traffic across both `vm-web1` and `vm-web2` | PASS |
| **Failover Test (`vm-web1` Stopped)** | `http://4.246.239.103` (`pip-lb`) | Zero downtime; 100% of traffic automatically served by `vm-web2` | PASS |
| **Telemetry & Health Monitoring** | Azure Monitor (`vm-web1` & `lb-web`) | Captured CPU stress spike (~18%) and Health Probe Status (`23.26k` Sum / `250` Count) | PASS |
