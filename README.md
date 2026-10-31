# AWS VPC Peering – Private Network Connectivity

**Project:** AWS VPC Peering with private EC2 instances and a Bastion host  
**Region used in this lab:** Mumbai (`ap-south-1`) — verify this matches your actual deployment.

## 1. Project Overview

This project connects two non-overlapping Amazon VPCs using an AWS VPC Peering connection. Each VPC contains a private application server. A public Bastion host in VPC A is used to administer both private instances over SSH.

### Objectives
- Build two VPCs with non-overlapping IPv4 CIDR blocks.
- Configure public and private subnets, route tables and an Internet Gateway.
- Deploy one public Bastion and two private Amazon Linux EC2 instances.
- Configure security groups with restricted SSH and ICMP ingress.
- Establish VPC Peering and configure routes in both directions.
- Verify SSH access through the Bastion and private IP connectivity.
- Demonstrate an ICMP security-group failure and recovery.

### Architecture

```text
Windows administrator
       |
       | SSH TCP/22 to Bastion public IPv4
       v
Internet Gateway
       |
VPC A: 10.10.0.0/16
  Public subnet: 10.10.1.0/24
    Bastion: 10.10.1.10 (public IPv4 assigned)
       |
       +---- SSH ProxyCommand / forwarded TCP
       |
  Private subnet: 10.10.2.0/24
    App-A: 10.10.2.10
       |
       | VPC Peering: private IPv4 traffic
       |
VPC B: 10.20.0.0/16
  Private subnet: 10.20.1.0/24
    App-B: 10.20.1.10
```

**Screenshot — Architecture diagram**

> [Insert architecture diagram here]

## 2. IP Addressing Plan

| Resource | Name | CIDR / Private IPv4 |
|---|---|---|
| VPC A | `ewb-vpc-a` | `10.10.0.0/16` |
| Public subnet A | `ewb-a-public` | `10.10.1.0/24` |
| Private subnet A | `ewb-a-private` | `10.10.2.0/24` |
| Bastion | `ewb-bastion` | `10.10.1.10` |
| App-A | `ewb-app-a` | `10.10.2.10` |
| VPC B | `ewb-vpc-b` | `10.20.0.0/16` |
| Private subnet B | `ewb-b-private` | `10.20.1.0/24` |
| App-B | `ewb-app-b` | `10.20.1.10` |

The VPC CIDRs do not overlap, which is required for this VPC Peering design.

## 3. AWS Resources

| Resource | Name |
|---|---|
| VPC A | `ewb-vpc-a` |
| VPC B | `ewb-vpc-b` |
| Internet Gateway | `ewb-igw-a` |
| Public route table | `ewb-rt-a-public` |
| Private route table A | `ewb-rt-a-private` |
| Private route table B | `ewb-rt-b-private` |
| Bastion security group | `ewb-sg-bastion` |
| App-A security group | `ewb-sg-app-a` |
| App-B security group | `ewb-sg-app-b` |
| Peering connection | `ewb-a-b-peer` |
| EC2 key pair | `ewb-lab` |

Instances: Amazon Linux 2023, x86_64, small lab instance type. Confirm the actual instance type and Region in the AWS Console before publishing.

## 4. VPC and Subnet Design

### VPC A
- Name: `ewb-vpc-a`
- IPv4 CIDR: `10.10.0.0/16`
- Public subnet: `ewb-a-public` — `10.10.1.0/24`
- Private subnet: `ewb-a-private` — `10.10.2.0/24`
- Internet Gateway: `ewb-igw-a`, attached to VPC A.
- Auto-assign public IPv4: enabled for public subnet A; disabled for private subnet A.

### VPC B
- Name: `ewb-vpc-b`
- IPv4 CIDR: `10.20.0.0/16`
- Private subnet: `ewb-b-private` — `10.20.1.0/24`
- Auto-assign public IPv4: disabled.
- No Internet Gateway route is configured for VPC B.

**Screenshot — VPC list and CIDR blocks**

> [Insert screenshot showing both VPC names and IPv4 CIDRs]

**Screenshot — Subnet configuration**

> [Insert screenshot showing all three subnets, CIDRs and Availability Zones]

**Screenshot — Internet Gateway attachment**

> [Insert screenshot showing `ewb-igw-a` attached to `ewb-vpc-a`]

## 5. Route Tables

VPC Peering does not automatically add routes. Each side needs a route to the other VPC CIDR. Return routes are necessary too.

### `ewb-rt-a-public` — associated with `ewb-a-public`

| Destination | Target |
|---|---|
| `10.10.0.0/16` | `local` |
| `0.0.0.0/0` | `ewb-igw-a` |
| `10.20.0.0/16` | `ewb-a-b-peer` (`pcx-...`) |

### `ewb-rt-a-private` — associated with `ewb-a-private`

| Destination | Target |
|---|---|
| `10.10.0.0/16` | `local` |
| `10.20.0.0/16` | `ewb-a-b-peer` (`pcx-...`) |

### `ewb-rt-b-private` — associated with `ewb-b-private`

| Destination | Target |
|---|---|
| `10.20.0.0/16` | `local` |
| `10.10.0.0/16` | `ewb-a-b-peer` (`pcx-...`) |

`pcx-...` is a placeholder for the actual VPC Peering connection ID. Do not type the dots literally.

**Screenshot — Public route table A**

> [Insert screenshot showing routes and subnet association]

**Screenshot — Private route table A**

> [Insert screenshot showing routes and subnet association]

**Screenshot — Private route table B**

> [Insert screenshot showing routes and subnet association]

## 6. VPC Peering

- Connection name: `ewb-a-b-peer`
- Requester: `ewb-vpc-a` (`10.10.0.0/16`)
- Accepter: `ewb-vpc-b` (`10.20.0.0/16`)
- Same AWS account and Region.
- Required state: **Active**.

Workflow:
1. Create the peering request from VPC A to VPC B.
2. Accept the request.
3. Confirm status is `Active`.
4. Add the routes listed in Section 5.
5. Verify traffic using private IPv4 addresses.

**Screenshot — Active VPC Peering connection**

> [Insert screenshot showing connection name, requester, accepter and Active status]

## 7. Security Groups

Security groups are stateful virtual firewalls attached to EC2 network interfaces. The lab retained the default allow-all outbound rule.

### `ewb-sg-bastion`

| Direction | Type | Protocol / Port | Source |
|---|---|---|---|
| Inbound | SSH | TCP 22 | Administrator's current public IPv4 `/32` |
| Outbound | All traffic | All | `0.0.0.0/0` |

Restrict Bastion SSH to your own current public IP. If the public IP changes, update the inbound rule. Avoid opening SSH to `0.0.0.0/0`.

### `ewb-sg-app-a`

| Direction | Type | Protocol / Port | Source |
|---|---|---|---|
| Inbound | SSH | TCP 22 | `10.10.1.10/32` |
| Inbound | ICMP Echo Request | ICMPv4 type 8, code 0 | `10.20.1.10/32` |
| Outbound | All traffic | All | `0.0.0.0/0` |

### `ewb-sg-app-b`

| Direction | Type | Protocol / Port | Source |
|---|---|---|---|
| Inbound | SSH | TCP 22 | `10.10.1.10/32` |
| Inbound | ICMP Echo Request | ICMPv4 type 8, code 0 | `10.10.2.10/32` |
| Outbound | All traffic | All | `0.0.0.0/0` |

The private instances accept SSH from the Bastion's **private** IP, not its public IP. App-A and App-B have no public IPv4 addresses.

**Screenshot — Bastion security group inbound rules**

> [Insert screenshot showing SSH limited to administrator public IP `/32`]

**Screenshot — App-A security group inbound rules**

> [Insert screenshot showing SSH and ICMP source rules]

**Screenshot — App-B security group inbound rules**

> [Insert screenshot showing SSH and ICMP source rules]

## 8. EC2 Instances

| Instance | Subnet | Private IPv4 | Public IPv4 |
|---|---|---|---|
| Bastion (`ewb-bastion`) | `ewb-a-public` | `10.10.1.10` | AWS-assigned |
| App-A (`ewb-app-a`) | `ewb-a-private` | `10.10.2.10` | None |
| App-B (`ewb-app-b`) | `ewb-b-private` | `10.20.1.10` | None |

All three instances use Amazon Linux 2023 and the `ewb-lab` key pair.

**Screenshot — EC2 instance overview**

> [Insert screenshot showing all three instances, state and private IPs]

**Screenshot — Bastion instance networking details**

> [Insert screenshot showing Bastion subnet, private/public IP and security group]

**Screenshot — App-A instance networking details**

> [Insert screenshot showing App-A private IP, subnet and security group]

**Screenshot — App-B instance networking details**

> [Insert screenshot showing App-B private IP, subnet and security group]

## 9. SSH from Windows PowerShell

The private key was stored locally at `$HOME\\.ssh\\ewb-lab.pem`. Never upload this PEM file to GitHub or copy it to the Bastion.

### Prepare the key (if needed)

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.ssh"
Move-Item "$HOME\Downloads\ewb-lab.pem" "$HOME\.ssh\ewb-lab.pem"
icacls "$HOME\.ssh\ewb-lab.pem" /inheritance:r
icacls "$HOME\.ssh\ewb-lab.pem" /grant:r "$($env:USERNAME):(R)"
Test-Path "$HOME\.ssh\ewb-lab.pem"
```

Expected final output: `True`. If the key is already in `.ssh`, do not run `Move-Item` again.

### Connect to Bastion

Replace `BASTION_PUBLIC_IP` with the current public IPv4 shown in EC2:

```powershell
ssh -o IdentitiesOnly=yes -i "$HOME\.ssh\ewb-lab.pem" ec2-user@BASTION_PUBLIC_IP
```

### Connect to App-A through Bastion

```powershell
ssh -o IdentitiesOnly=yes -o ProxyCommand="ssh -o IdentitiesOnly=yes -i $HOME\.ssh\ewb-lab.pem -W %h:%p ec2-user@BASTION_PUBLIC_IP" -i "$HOME\.ssh\ewb-lab.pem" ec2-user@10.10.2.10
```

### Connect to App-B through Bastion

```powershell
ssh -o IdentitiesOnly=yes -o ProxyCommand="ssh -o IdentitiesOnly=yes -i $HOME\.ssh\ewb-lab.pem -W %h:%p ec2-user@BASTION_PUBLIC_IP" -i "$HOME\.ssh\ewb-lab.pem" ec2-user@10.20.1.10
```

ProxyCommand makes the local SSH client connect to the Bastion first, then forward the TCP connection to the private instance. The private key stays on the Windows computer.

**Screenshot — SSH login to Bastion**

> [Insert terminal screenshot]

**Screenshot — SSH login to App-A through Bastion**

> [Insert terminal screenshot]

**Screenshot — SSH login to App-B through Bastion**

> [Insert terminal screenshot]

## 10. Connectivity Validation Commands

Run Linux commands after connecting to the relevant EC2 instance.

### Check host and IP

```bash
hostname
hostname -I
ip -4 addr
ip route
```

Expected private IP:
- App-A: `10.10.2.10`
- App-B: `10.20.1.10`

### App-A to App-B ping (run on App-A)

```bash
ping -c 4 10.20.1.10
```

Expected after routes and ICMP rules are configured: replies from App-B.

### App-B to App-A ping (run on App-B)

```bash
ping -c 4 10.10.2.10
```

Expected: replies from App-A.

### Check SSH port reachability (run on Bastion)

```bash
nc -vz -w 5 10.10.2.10 22
nc -vz -w 5 10.20.1.10 22
```

A successful `nc` connection verifies TCP port reachability; it does not prove SSH key authentication.

### Troubleshooting order

1. Confirm the EC2 instance is `Running` and status checks pass.
2. Verify the destination private IP.
3. Verify the correct security group is attached.
4. Verify SSH/ICMP inbound sources and ports.
5. Verify each subnet's route-table association.
6. Verify routes to the remote VPC CIDR on both sides.
7. Verify peering status is `Active`.
8. Check network ACLs if routes and security groups are correct.
9. For SSH authentication errors, confirm the instance uses the expected key pair and the local key file is correct.

**Screenshot — App-A to App-B ping success**

> [Insert terminal screenshot showing successful `ping -c 4 10.20.1.10`]

**Screenshot — App-B to App-A ping success**

> [Insert terminal screenshot showing successful `ping -c 4 10.10.2.10`]

**Screenshot — Bastion TCP port checks**

> [Insert terminal screenshot showing successful `nc` checks on port 22]

## 11. Controlled ICMP Failure and Recovery

This test demonstrates that the security group can block ICMP while SSH remains allowed.

### Procedure

1. Confirm App-A can ping App-B.
2. In AWS Console, open `ewb-sg-app-b` → Inbound rules → Edit.
3. Temporarily remove only the ICMP Echo Request rule with source `10.10.2.10/32`.
4. Keep the SSH rule from `10.10.1.10/32` unchanged.
5. From App-A, run:

```bash
ping -c 4 10.20.1.10
```

6. Expected: ping fails because App-B no longer allows ICMP Echo Requests from App-A.
7. Verify SSH to App-B through the Bastion still works.
8. Restore the ICMP rule on `ewb-sg-app-b`:

| Setting | Value |
|---|---|
| Type | All ICMP - IPv4 (or ICMP Echo Request) |
| Source | `10.10.2.10/32` |

9. Save the rule.
10. From App-A, run again:

```bash
ping -c 4 10.20.1.10
```

Expected: successful replies after restoring the rule.

### Interpretation

- Ping failure with the ICMP rule removed demonstrates security-group filtering.
- SSH success during the ICMP block demonstrates that the separate SSH rule remains effective.
- Ping success after restoring ICMP demonstrates recovery.

Do not claim a test succeeded unless the terminal evidence actually shows the expected result.

**Screenshot — Controlled ICMP failure**

> [Insert terminal screenshot showing App-A ping failing while the App-B ICMP rule is removed]

**Screenshot — SSH still works during ICMP block**

> [Insert terminal screenshot showing successful SSH to App-B while ICMP is blocked]

**Screenshot — ICMP rule restored**

> [Insert AWS Console screenshot showing the restored ICMP inbound rule]

**Screenshot — Connectivity restored**

> [Insert terminal screenshot showing successful App-A to App-B ping after restoration]

## 12. Results and Conclusion

The intended final state is:
- Two non-overlapping VPCs connected by an active VPC Peering connection.
- One public Bastion in VPC A.
- Two private application instances without public IPv4 addresses.
- SSH administration to both private instances through the Bastion.
- Bidirectional private IP connectivity between App-A and App-B.
- Correct routes in both VPCs.
- Restricted SSH and ICMP security-group rules.
- Demonstrated ICMP failure and recovery.

**Conclusion:** This lab demonstrates that VPC Peering provides private network connectivity between VPCs, but it does not automatically configure routing or security permissions. Routes are required on both sides, and security groups independently control permitted traffic.

## 13. Cost Awareness and Cleanup

EC2 runtime, EBS storage and public IPv4 addresses may incur charges. Check current AWS pricing for the selected Region and instance types. Save evidence and documentation before cleanup.

Suggested cleanup sequence:
1. Terminate the three EC2 instances.
2. Check for and remove unused EBS volumes and other billable resources.
3. Delete the VPC Peering connection.
4. Delete lab security groups after they are detached from network interfaces.
5. Remove route-table subnet associations, then delete lab route tables.
6. Detach and delete `ewb-igw-a`.
7. Delete the lab subnets.
8. Delete both lab VPCs.
9. Verify that no lab resources remain.

AWS may prevent deletion while dependencies exist. Follow the console's dependency messages.

**Screenshot — Cleanup evidence**

> [Insert screenshot showing terminated instances and/or final cleanup state]

## 14. GitHub Repository Setup

Suggested repository name:

`aws-vpc-peering-private-connectivity`

Suggested description:

> Private connectivity between two AWS VPCs using VPC Peering, private EC2 instances, route tables, security groups and a Bastion host.

### Push from PowerShell

Run from the project directory after saving this file as `README.md` and adding your screenshots:

```powershell
git init
git add README.md .gitignore screenshots
git status
git commit -m "Add AWS VPC peering project documentation and evidence"
git branch -M main
git remote add origin https://github.com/YOUR-GITHUB-USERNAME/aws-vpc-peering-private-connectivity.git
git push -u origin main
```

Replace `YOUR-GITHUB-USERNAME` with your GitHub username. If a remote already exists, do not add it again.

Before pushing, inspect tracked files:

```powershell
git status
git ls-files
```

Never commit `ewb-lab.pem`, AWS credentials, tokens, `.env` files or other secrets. If a secret is committed, deleting it in a later commit is not enough: revoke/rotate it and clean the repository history.

## 15. Resume / Portfolio Summary

**AWS VPC Peering – Private Network Connectivity**

Designed and implemented private IPv4 connectivity between two AWS VPCs using VPC Peering. Configured public and private subnets, route tables, an Internet Gateway, security groups and three Amazon Linux EC2 instances. Used a Bastion host and SSH forwarding to administer private instances. Validated bidirectional private connectivity and demonstrated security-group traffic control through a controlled ICMP failure and recovery.

Use this summary only if the final AWS configuration and screenshots in the repository support every claim.

## 16. Screenshot Index

Add each image to the path shown below. The screenshot headings above explain what each image should prove.

| Screenshot title | Suggested path |
|---|---|
| VPC names and CIDRs | `screenshots/vpc-cidrs.png` |
| Subnets and CIDRs | `screenshots/subnets.png` |
| Internet Gateway attachment | `screenshots/internet-gateway.png` |
| Bastion security group | `screenshots/bastion-sg.png` |
| App-A security group | `screenshots/app-a-sg.png` |
| App-B security group | `screenshots/app-b-sg.png` |
| EC2 instance overview | `screenshots/instances-overview.png` |
| Active VPC Peering | `screenshots/peering-active.png` |
| Public route table A | `screenshots/public-route-table-a.png` |
| Private route table A | `screenshots/private-route-table-a.png` |
| Private route table B | `screenshots/private-route-table-b.png` |
| SSH to Bastion | `screenshots/ssh-bastion.png` |
| SSH to App-A | `screenshots/ssh-app-a.png` |
| SSH to App-B | `screenshots/ssh-app-b.png` |
| App-A to App-B ping | `screenshots/app-a-to-app-b.png` |
| App-B to App-A ping | `screenshots/app-b-to-app-a.png` |
| Controlled ICMP failure | `screenshots/icmp-failure.png` |
| SSH works during ICMP block | `screenshots/ssh-during-icmp-block.png` |
| ICMP restored and ping succeeds | `screenshots/icmp-restored.png` |
| Cleanup evidence | `screenshots/cleanup.png` |
