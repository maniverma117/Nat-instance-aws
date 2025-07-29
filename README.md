# Custom NAT Instance Setup on Amazon Linux 2 (Graviton Compatible)

This guide describes how to create and configure a custom NAT instance on AWS using Amazon Linux 2, including Graviton (ARM-based) instance support.

---

## ✅ Prerequisites

- VPC with:
  - At least one **public subnet** (with Internet Gateway)
  - At least one **private subnet** (no direct Internet Gateway access)
- **Amazon Linux 2 (ARM64 or x86_64)** AMI
- **Elastic IP** for the NAT instance

---

## 🚀 Step-by-Step Instructions

### 1. Launch NAT Instance

- AMI: Amazon Linux 2 (64-bit ARM or x86)
- Instance Type: `t4g.small`, `t4g.medium`, or `t3.micro`
- Subnet: Public subnet
- Auto-assign Public IP: **Enabled**
- Security Group:
  - **Inbound**:
    - SSH (22) from your IP
    - (Optional) ICMP from private subnet
  - **Outbound**:
    - Allow **All traffic**
- Attach Elastic IP

> ✅ **Disable Source/Destination Check** after launch:
>
> Go to EC2 → Instance → Actions → Networking → **Change Source/Dest Check → Disable**

---

### 2. Connect and Configure NAT

SSH into the NAT instance and run:

```bash
# Install iptables service
sudo yum install iptables-services -y

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Identify network interface (usually ens5, eth0, or ensX)
ip route | grep default

# Apply NAT masquerade rule (replace ens5 with your interface name if different)
sudo iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE

# Save and enable iptables
sudo service iptables save
sudo systemctl enable iptables
sudo systemctl start iptables
````

---

### 3. Update Private Subnet Route Table

Go to **VPC → Route Tables**, select the one associated with your **private subnet**, and add:

| Destination | Target          |
| ----------- | --------------- |
| 0.0.0.0/0   | NAT Instance ID |

---

### 4. Test from a Private Instance

* Launch a new EC2 in your **private subnet** (no public IP)
* Use Session Manager or Bastion to connect
* Run:

```bash
ping 8.8.8.8
curl ifconfig.me
```

You should see the public IP of your NAT instance.

---

## 🔍 Troubleshooting

### 1. Check IP Forwarding

```bash
cat /proc/sys/net/ipv4/ip_forward
# Should be 1
```

### 2. Check NAT Rules

```bash
sudo iptables -t nat -L -n -v
# Should show MASQUERADE on correct interface
```

### 3. Watch Traffic

```bash
sudo tcpdump -i ens5 icmp or port 80 or port 443
```

### 4. Ensure Security Groups and NACLs

* Allow all **outbound** traffic
* Inbound:

  * NAT: allow SSH, ICMP (optional)
  * Private instance: allow HTTP/HTTPS outbound

---

## 🧠 Notes

* This setup replaces the deprecated AWS-managed NAT AMI.
* Using Graviton (e.g., `t4g.medium`) is recommended for higher throughput and lower cost.
* You can save the configured NAT instance as a **custom AMI** for reuse.

---

## 📌 License

MIT License – use at your own risk. Test before using in production.
