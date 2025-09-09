# AWS Networking Lab — Hybrid Networking (Site‑to‑Site VPN + Client VPN)

In this lab you’ll build a **hybrid network** with **two VPN styles**:

1) **Site‑to‑Site (IPsec) VPN** between AWS and your **on‑prem VM** (strongSwan/Openswan on your laptop).  
2) **AWS Client VPN** (SSL) so users can dial in securely from laptops.

You’ll test connectivity from **on‑prem → AWS private subnet** and compare operational differences between **IPsec S2S** and **SSL Client VPN**.

> ⚠️ **Costs**: Uses **NAT Gateway**, **Client VPN endpoint**, EC2 instances, and a **VGW/VPN**. These are **paid** resources. Keep the lab short and **delete the stack** when done.

---

## 🗺️ Architecture Diagram

```mermaid
flowchart LR
  subgraph OnPrem["On‑Prem VM (Laptop)"]
    CGW[(strongSwan/Openswan VM<br/>Public IP = OnPremPublicIp)]
    ONP_SUB[On‑Prem Subnet<br/>192.168.100.0/24]
  end

  subgraph AWS["AWS VPC 10.40.0.0/16"]
    PUB["Public 10.40.1.0/24<br/>IGW + NAT GW + Bastion"]
    PRIV["Private 10.40.2.0/24<br/>App EC2 (no public IP)"]
    BAST[(Bastion EC2)]
    APP[(Private App EC2)]
    IGW[Internet Gateway]
    NAT[NAT Gateway]
    VGW[VPN Gateway]
    CVPN[(Client VPN Endpoint<br/>172.16.0.0/22)]
  end

  CGW == IPsec S2S == VGW
  CVPN --- PRIV
  BAST --- PUB
  APP --- PRIV
  PUB --- IGW
  PUB --- NAT
```

---

## 📦 What the stack creates

- **VPC 10.40.0.0/16** with:
  - **Public subnet** `10.40.1.0/24` (0.0.0.0/0 → IGW)
  - **Private subnet** `10.40.2.0/24` (0.0.0.0/0 → **NAT Gateway**)
- **EC2 Instances**
  - **Bastion** (public subnet) — SSH entry point
  - **Private App** (private subnet) — HTTP server (no public IP)
- **VPN Gateway (VGW)** attached to the VPC
- **Customer Gateway (CGW)** (static routing) — uses your **OnPremPublicIp**
- **VPN Connection** (static) with a route to your **OnPrem subnet** (e.g., `192.168.100.0/24`)
- **Client VPN Endpoint** (SSL) — requires ACM certs (you’ll import)
  - Association to the **public subnet**
  - Authorization to reach `10.40.0.0/16`
- **Security Groups** permitting SSH/HTTP appropriately

---

## 🔧 Parameters

- `KeyName` — EC2 key pair for SSH  
- `YourIpCidr` — your IP in CIDR (e.g., `203.0.113.10/32`) to allow SSH to Bastion  
- `OnPremPublicIp` — the public IP of your laptop VM (strongSwan/Openswan)  
- `OnPremSubnetCidr` — your on‑prem subnet (e.g., `192.168.100.0/24`)  
- `InstanceType` — default `t3.micro`  
- `ClientVpnCidr` — client address pool, default `172.16.0.0/22`  
- `ServerCertificateArn` — ACM ARN for the **Client VPN server** cert  
- `ClientRootCertArn` — ACM ARN for the **client root CA** (mutual auth)

> You can import self‑signed certs into ACM for testing.

---

## 🚀 Deploy

1. **Create/import ACM certificates** (region = where you deploy):
   - Generate a quick PKI (OpenSSL):
     ```bash
     # Root CA
     openssl genrsa -out rootCA.key 2048
     openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 365 -out rootCA.crt -subj "/CN=lab-root"
     # Server cert (CN must match any name; used only by Client VPN)
     openssl genrsa -out server.key 2048
     openssl req -new -key server.key -out server.csr -subj "/CN=lab-server"
     openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out server.crt -days 365 -sha256
     # Client cert
     openssl genrsa -out client.key 2048
     openssl req -new -key client.key -out client.csr -subj "/CN=lab-client"
     openssl x509 -req -in client.csr -CA rootCA.crt -CAkey rootCA.key -CAcreateserial -out client.crt -days 365 -sha256
     ```
   - **ACM → Import**:
     - **ServerCertificateArn**: `server.crt` + `server.key` (and **rootCA.crt** as chain).
     - **ClientRootCertArn**: import **rootCA.crt** (no key).

2. **AWS Console → CloudFormation → Create stack** → *With new resources*.  
   Upload `cloudformation/hybrid-vpn.yaml`. Set parameters (`KeyName`, `YourIpCidr`, `OnPremPublicIp`, `OnPremSubnetCidr`, `ServerCertificateArn`, `ClientRootCertArn`). Create.

3. Wait for **CREATE_COMPLETE** and open **Outputs**:
   - `BastionPublicIP` — SSH entry point
   - `AppPrivateIP` — the private IP you’ll test from on‑prem
   - `ClientVpnEndpointId` — for Client VPN downloads
   - `VpnConnectionId` — for tunnel details

---

## 🧪 Lab Steps

### Part A — Site‑to‑Site (IPsec) with strongSwan
1. On your **on‑prem VM** (Ubuntu/Debian is fine), install strongSwan:
   ```bash
   sudo apt update && sudo apt install -y strongswan
   ```
2. In AWS Console → **VPC → Site‑to‑Site VPN Connections → your connection → Download configuration**.  
   Choose **Vendor: Generic** / **Platform: strongSwan** (or “Other”). This gives example `ipsec.conf`/`ipsec.secrets`.
3. Edit `/etc/ipsec.conf` and `/etc/ipsec.secrets` to match:
   - **Local (left)** = your on‑prem public IP (**OnPremPublicIp**) & subnet (**OnPremSubnetCidr**)
   - **Remote (right)** = AWS tunnel outside IP + VPC CIDR `10.40.0.0/16`
   - Example policy (AES256/SHA1/DH14) — follow the downloaded config.
4. Enable IP forwarding on your VM:
   ```bash
   echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
   sudo sysctl -w net.ipv4.ip_forward=1
   ```
5. Start the tunnel:
   ```bash
   sudo ipsec restart
   sudo ipsec statusall
   ```
6. Test from on‑prem VM to the **private app**:
   ```bash
   ping -c 3 <AppPrivateIP>
   curl http://<AppPrivateIP>
   ```
   If curl resolves, you’re traversing the IPsec tunnel into the AWS **private subnet**.

> If you cannot get a public IP on your laptop VM, you can port‑forward IPsec on your router or host the “on‑prem” strongSwan on a **cloud VM with a public IP** just for demo.

### Part B — AWS Client VPN (SSL)
1. **Associate** the Client VPN endpoint to the VPC (the stack does the first association).  
2. **Download** the Client VPN configuration file (VPC → **Client VPN Endpoints** → **Download client configuration**).
3. Append your client cert and key to the `.ovpn`:
   ```
   <cert>
   # paste contents of client.crt
   </cert>
   <key>
   # paste contents of client.key
   </key>
   ```
4. Connect using your OpenVPN client (or AWS VPN Client).  
5. Test reachability:
   ```bash
   ping -c 3 <AppPrivateIP>
   curl http://<AppPrivateIP>
   ```

---

## 🧭 Console checkpoints

- **VPC → Route tables**: private RT has `0.0.0.0/0 → NAT` and route to **OnPremSubnetCidr → VGW**.  
- **Site‑to‑Site VPN**: both **tunnels** should show **UP** (or at least one).  
- **Client VPN**: endpoint **Associated**, **Authorization** to `10.40.0.0/16` enabled.  
- **EC2 → Security Groups**: SSH restricted to **YourIpCidr**.  
- **EC2**: App instance has **only private IP**.

---

## 🆘 Troubleshooting

- **No tunnel**: Check your router/NAT supports IPsec or try an external VM with a public IP for strongSwan.  
- **Ping works, HTTP fails**: SG or NACL rules might block port 80; verify SG allows `80` from VPC/Client CIDRs.  
- **Client VPN connects but can’t reach VPC**: Confirm **authorization rule** and **route** to `10.40.0.0/16`.  
- **ACM import fails**: Ensure PEM formatting and that chain includes **rootCA.crt** for the server cert.
- **Private instance can’t `dnf`/`yum`**: NAT Gateway must be **Available** and private RT default to **NAT**.

---

## 🧹 Cleanup

CloudFormation → select the stack → **Delete**.  
Remove any imported ACM certs and downloaded client config on your machine if not needed.

---

## 📁 Repo Structure

```
cloudformation/
  hybrid-vpn.yaml
docs/
  architecture.md
.github/workflows/
  cfn-validate.yml
README.md
LICENSE
.gitignore
```
