1. Outbound Traffic via pfSense (NAT Setup)
Since Linode doesn’t provide NAT, pfSense must act as a NAT gateway for your private Ubuntu 24 instances.

Step 1: Configure NAT on pfSense

Log in to the pfSense Web UI.
Navigate to Firewall > NAT > Outbound.
Set the mode to Manual Outbound NAT.
Add a rule:
Interface: WAN
Source: 172.16.10.0/24
Destination: Any
Translation: Use WAN Address (172.236.175.231)
Click Save & Apply Changes.

2. Configure Ubuntu VM to Use pfSense as Default Gateway
Each private instance must route traffic through pfSense.

Step 1: Set Default Gateway on Ubuntu
Run:

sudo ip route add default via private subnet

To persist the route, update Netplan, look at Netplan.yml file:

sudo nano /etc/netplan/50-cloud-init.yaml

```
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 172.16.10.2/24
      gateway4: 172.16.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

					
Apply changes:
sudo netplan apply

3. Inbound Traffic Routing (Port Forwarding to Private Instances)
You need to configure pfSense to route external traffic to backend instances.

Step 1: Configure Port Forwarding on pfSense
Navigate to Firewall > NAT > Port Forward.
Click Add:
Interface: WAN
Protocol: TCP
Destination: WAN Address private subnet
Destination Port: (e.g., 443 for HTTPS, 22 for SSH)
Redirect Target IP: private subnet
Redirect Target Port: (same as destination)
Click Save & Apply Changes.

Step 2: Allow Traffic in pfSense Firewall
Go to Firewall > Rules > WAN.
Click Add:
Action: Pass
Interface: WAN
Protocol: TCP
Source: Any
Destination: private subnet
Destination Port: (same as port forwarding)
Click Save & Apply Changes.

Step 3: Allow Traffic on Ubuntu Firewall
Run on private subnet:

```sudo ufw allow 22/tcp   # SSH```
```sudo ufw allow 80/tcp   # HTTP```
```sudo ufw allow 443/tcp  # HTTPS```
```sudo ufw enable```

4. Ensure VPN Traffic Works with AWS
Since you’ve established an IPsec tunnel between pfSense and AWS:

Step 1: Update Firewall Rules for VPN Traffic
In pfSense:

Navigate to Firewall > Rules > IPsec.
Add a rule to allow all traffic:
Action: Pass
Protocol: Any
Source: 172.16.10.0/24
Destination: AWS subnet (e.g., 10.0.0.0/16)
Click Save & Apply Changes.

Step 2: Verify Routing from private subnet to AWS
On private VM, try:

```ping <AWS-Private-IP>```
```traceroute <AWS-Private-IP>```

If it fails:

Ensure AWS has a route back to pFsense instance.
Check AWS security groups allow traffic from pFsense instance public IP.

5. Testing & Verification
Outbound NAT Test (Ubuntu)
Run on pvt subnet:

```curl ifconfig.me```

Expected result: (pfSense’s public IP).

Inbound Routing Test (From External Machine)
Try SSH:

```ssh user@pFsenseIP```

Expected: You should reach private subnet.

VPN Test (Ubuntu → AWS)
Run:

```ping <AWS-Private-IP>```

Expected: Successful ping.









