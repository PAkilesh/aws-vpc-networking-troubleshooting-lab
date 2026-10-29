# AWS VPC Networking & Troubleshooting Lab

## Project Overview

A hands-on AWS networking project focused on designing a custom Virtual Private Cloud (VPC), configuring public and private subnets, and troubleshooting connectivity issues across AWS networking components.

The lab simulates real-world Cloud Support Engineer scenarios involving Security Groups, Network ACLs, route tables, internet connectivity, and secure SSH access to private EC2 instances.

**AWS Region:** Asia Pacific (Mumbai) — `ap-south-1`

## Architecture

```text
                  Internet
                     |
              Internet Gateway
                     |
              Custom AWS VPC
                10.0.0.0/16
                     |
          +----------+----------+
          |                     |
     Public Subnet         Private Subnet
      10.0.2.0/24           10.0.1.0/24
          |                     |
     Public EC2             Private EC2
      10.0.2.86              10.0.1.152
      Nginx Web Server       SSH Service
          |                     |
      Public IP:            No Public IP
     13.235.99.186
          |
       SSH Jump Host
          |
          +------ SSH ------> Private EC2
```

## AWS Services and Technologies

- Amazon VPC
- Amazon EC2 — Amazon Linux 2023
- Public and Private Subnets
- Internet Gateway
- Route Tables
- Security Groups
- Network Access Control Lists (NACLs)
- SSH and SSH ProxyJump
- Nginx Web Server
- Linux Networking and Troubleshooting
- Windows PowerShell

## Infrastructure Configuration

| Component | Configuration |
|---|---|
| VPC | `10.0.0.0/16` |
| Public Subnet | `10.0.2.0/24` |
| Private Subnet | `10.0.1.0/24` |
| Availability Zone | `ap-south-1a` |
| Public EC2 | Amazon Linux 2023, t3.micro |
| Private EC2 | Amazon Linux 2023, t3.micro |
| Web Server | Nginx |
| Public Route | `0.0.0.0/0` → Internet Gateway |
| Private Route | `10.0.0.0/16` → local |
| Private Instance Access | SSH through public EC2 jump host |

## Implementation

### 1. Custom VPC and Subnets

Created a custom VPC with CIDR block `10.0.0.0/16`.

Configured two subnets:

- Public subnet: `10.0.2.0/24`
- Private subnet: `10.0.1.0/24`

Attached an Internet Gateway and configured a default internet route for the public subnet.

The private subnet was configured without a NAT Gateway or internet default route.

### 2. Public EC2 and Nginx

Launched an Amazon Linux 2023 EC2 instance in the public subnet.

Installed and started Nginx:

```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

Verified HTTP service locally:

```bash
curl -I http://localhost
```

Verified public HTTP connectivity using Windows PowerShell:

```powershell
curl.exe -I http://13.235.99.186
```

**Result:** `HTTP/1.1 200 OK`

### 3. Security Group Troubleshooting

**Incident:** The public website became inaccessible while the EC2 instance and Nginx remained healthy.

**Simulation:** Removed the inbound HTTP rule permitting TCP port 80.

**Troubleshooting:**
- Verified EC2 instance health.
- Confirmed Nginx was running.
- Confirmed local HTTP requests returned `200 OK`.
- Observed external HTTP connection timeouts.
- Reviewed inbound Security Group rules.

**Root Cause:** Missing inbound HTTP permission.

**Resolution:** Restored TCP port 80 access and confirmed successful HTTP connectivity.

### 4. Route Table Troubleshooting

**Incident:** The public EC2 website became unreachable.

**Simulation:** Temporarily removed the public route table's default route to the Internet Gateway.

**Root Cause:** Missing `0.0.0.0/0` route to the Internet Gateway.

**Resolution:** Restored the default route and verified HTTP access.

**Learning:** A healthy application and correctly configured Security Group are not sufficient when the subnet has no valid internet route.

### 5. Network ACL Troubleshooting

Created a custom Network ACL for the public subnet.

Configured inbound SSH and HTTP rules and outbound ephemeral-port permissions.

Simulated an HTTP connectivity failure by adding an explicit deny rule:

| Rule | Port | Action |
|---|---|---|
| 100 | SSH 22 | Allow |
| 105 | HTTP 80 | Deny |
| 110 | HTTP 80 | Allow |

Because NACL rules are processed in ascending rule-number order, rule 105 blocked HTTP traffic before rule 110 could allow it.

**Resolution:** Removed rule 105 and confirmed that the website returned `HTTP/1.1 200 OK`.

**Learning:** Network ACLs are stateless and support explicit allow and deny rules.

### 6. Private EC2 and SSH Jump Host

Launched a second Amazon Linux EC2 instance in the private subnet without a public IPv4 address.

Configured the private EC2 Security Group to allow SSH only from the public EC2 Security Group.

Initially, SSH connectivity from the public instance to the private instance failed.

Tested TCP port 22 connectivity:

```bash
timeout 8 bash -c 'echo > /dev/tcp/10.0.1.152/22' && echo "SSH PORT REACHABLE" || echo "SSH CONNECTION FAILED"
```

**Root Cause:** The public subnet's custom Network ACL did not permit outbound TCP port 22 traffic to the private subnet.

**Resolution:**
- Allowed outbound TCP 22 to `10.0.1.0/24`.
- Allowed inbound ephemeral-port return traffic from `10.0.1.0/24`.
- Retested connectivity successfully.

**Result:** `SSH PORT REACHABLE`

Connected to the private EC2 using SSH ProxyCommand through the public instance, keeping the private SSH key on the local Windows laptop.

Verified the private instance:

```bash
hostname
hostname -I
```

**Result:** `10.0.1.152`

### 7. Private Subnet Internet Isolation

From the private EC2 instance, inspected the Linux routing table:

```bash
ip route
```

Tested outbound HTTPS connectivity:

```bash
curl -I --connect-timeout 5 --max-time 10 https://www.amazon.com
```

**Result:** Connection timed out.

The private subnet had only a VPC-local route and no internet egress route through a NAT Gateway or other gateway.

This demonstrated the difference between private VPC connectivity and outbound internet access.

## Troubleshooting Summary

| Scenario | Root Cause | Resolution |
|---|---|---|
| Public website timeout | Security Group blocked HTTP | Restored inbound TCP 80 |
| Public website unreachable | Missing Internet Gateway route | Restored default route |
| HTTP blocked at subnet level | Explicit NACL deny rule | Removed deny rule |
| Public-to-private SSH failure | Missing outbound NACL TCP 22 rule | Allowed SSH to private subnet |
| Private EC2 internet timeout | No internet egress route | Verified expected private subnet isolation |

## Lab Screenshots

The following screenshots document the AWS VPC configuration and troubleshooting exercises.

![Screenshot 1](Screenshot%202026-10-29%20195750.png)

![Screenshot 2](Screenshot%202026-10-29%20195814.png)

![Screenshot 3](Screenshot%202026-10-29%20195854.png)

![Screenshot 4](Screenshot%202026-10-29%20195906.png)

![Screenshot 5](Screenshot%202026-10-29%20195948.png)

![Screenshot 6](Screenshot%202026-10-29%20200011.png)

![Screenshot 7](Screenshot%202026-10-29%20200140.png)

![Screenshot 8](Screenshot%202026-10-29%20200225.png)

![Screenshot 9](Screenshot%202026-10-29%20200307.png)

## Key Learnings

- Designing public and private AWS VPC networking.
- Understanding Security Groups versus Network ACLs.
- Diagnosing network timeouts using a layered troubleshooting approach.
- Understanding stateless NACL rules and ephemeral return ports.
- Securing private EC2 access through an SSH jump host.
- Troubleshooting Linux network connectivity.
- Verifying private subnet internet isolation.
- Documenting incidents, root causes, and resolutions.

## Cost Management

The lab avoids using a NAT Gateway to reduce unnecessary costs. EC2 instances and other billable resources should be terminated or deleted after completing the project.

## Author

**P. Akhilesh**

GitHub: https://github.com/PAkilesh

*Hands-on AWS networking and troubleshooting project developed to strengthen Cloud Support and Cloud Operations engineering skills.*
