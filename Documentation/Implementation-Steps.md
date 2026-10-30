# Implementation Steps — EWB VPC Peering

**Status:** Instructions only. No resources have been created by this package.

## 1. Plan and protect the account
Use a permitted AWS lab account, a single Region (example `ap-south-1`), and configure a budget. Add `Project=EWB-VPC-Peering` and `Student=<your name>` tags. Check current costs for EC2, EBS and public IPv4.

## 2. VPC A and public access
1. Create VPC-only `ewb-vpc-a` with `10.10.0.0/16`.
2. Create `ewb-a-public` (`10.10.1.0/24`) and `ewb-a-private` (`10.10.2.0/24`).
3. Create and attach `ewb-igw-a` to VPC A.
4. Create `ewb-rt-a-public`; explicitly associate public subnet; retain local route and add `0.0.0.0/0 -> igw-ID`.
5. Create `ewb-rt-a-private`; explicitly associate private subnet; retain local route, no internet default route.

## 3. VPC B
1. Create VPC-only `ewb-vpc-b` with `10.20.0.0/16`.
2. Create `ewb-b-private` (`10.20.1.0/24`), disable public IPv4 assignment.
3. Create `ewb-rt-b-private`; explicitly associate subnet; retain local route. No internet gateway or NAT gateway.

## 4. Security groups (inbound)
- `ewb-sg-bastion` in VPC A: TCP 22 from **your current public IPv4/32**.
- `ewb-sg-app-a` in VPC A: TCP 22 from `10.10.1.10/32`; ICMP IPv4 Echo request (type 8, code 0) from `10.20.1.10/32`.
- `ewb-sg-app-b` in VPC B: TCP 22 from `10.10.1.10/32`; ICMP IPv4 Echo request from `10.10.2.10/32`.
Keep default allow-all outbound security group rules and default allow-all NACLs for this lab. Never open SSH to `0.0.0.0/0`.

## 5. EC2 instances
Create RSA key pair `ewb-lab` locally, **never upload the private key to GitHub or EC2**. Launch three Amazon Linux 2023 instances (approved x86 size, e.g. `t3.micro`) using the same key:
- `ewb-bastion`: public subnet A, private `10.10.1.10`, public IPv4 enabled, bastion SG.
- `ewb-app-a`: private subnet A, private `10.10.2.10`, no public IPv4, App-A SG.
- `ewb-app-b`: private subnet B, private `10.20.1.10`, no public IPv4, App-B SG.
Wait for instance status checks. Record actual IDs and addresses.

## 6. SSH configuration (on YOUR laptop)
Move key to `~/.ssh/ewb-lab.pem` and run `chmod 400 ~/.ssh/ewb-lab.pem`. Save this in `~/.ssh/config`, replacing `BASTION_PUBLIC_IP`:

```sshconfig
Host ewb-bastion
    HostName BASTION_PUBLIC_IP
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

Connect `ssh ewb-bastion`, then `ssh ewb-app-a`. Inside App-A run `ping -c 4 10.20.1.10` **before peering** and save the failed baseline.

## 7. Peering and routes
Create `ewb-a-b-peer`: requester VPC A, accepter VPC B, same account and Region. Accept the request; verify **Active** and record `pcx-ID`.

Update routes (retain AWS local routes):

| Route table / subnet | Additional destination -> target |
|---|---|
| `ewb-rt-a-private` / `ewb-a-private` | `10.20.0.0/16 -> pcx-ID` |
| `ewb-rt-b-private` / `ewb-b-private` | `10.10.0.0/16 -> pcx-ID` |
| `ewb-rt-a-public` / `ewb-a-public` | `10.20.0.0/16 -> pcx-ID` (in addition to IGW default) |

The public subnet route is needed because the bastion opens SSH to App-B through ProxyJump. The reverse route in VPC B is required for return packets.

## 8. Tests — capture actual terminal output
- Before peering: from App-A `ping -c 4 10.20.1.10` -> expected no replies.
- After routes: from App-A `ping -c 4 10.20.1.10` -> expected replies.
- From App-B `ping -c 4 10.10.2.10` -> expected replies.
- From laptop `ssh ewb-app-b` -> expected shell via bastion. On private hosts run `hostname` and `hostname -I`.
- Temporarily delete **only** App-B inbound ICMP echo request from `10.10.2.10/32`, repeat App-A ping -> expected failure; restore the rule -> expected recovery.
- In EC2 details verify App-A and App-B have no public IPv4.

**Do not claim success until you observe it.** If a test fails, check instance health, actual IPs, subnet associations, peer status, both routes and security group rules.

## 9. Cleanup
After saving evidence: terminate all 3 EC2 instances; check EBS deletion; release any lab Elastic IP; remove peering routes, delete peering, security groups, subnets and custom route tables; detach/delete IGW; delete both VPCs; delete unused lab key pair and local key if no longer needed. Check for residual billable resources and capture cleanup evidence.

## 10. Submission
Fill in report with student name, batch, date, real AWS IDs, actual results and screenshot evidence. Upload README, architecture, documentation, report and screenshots to GitHub. Share repository link when EWB sends the form.
