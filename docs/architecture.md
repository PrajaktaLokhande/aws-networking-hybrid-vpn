# Architecture

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
