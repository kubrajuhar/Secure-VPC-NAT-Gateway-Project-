# Secure-VPC-NAT-Gateway-Project

AWS Networking & Secure Subnetting — Portfolio Project

1. Project Overview
This project demonstrates a foundational AWS networking build: a Virtual Private Cloud (VPC) with clearly separated public and private subnets, a NAT Gateway for secure outbound-only internet access from private resources, and security groups designed using group-to-group referencing rather than static IP ranges.
The goal was to build and validate every core networking primitive from scratch through the AWS Console, and to document each design decision including trade-offs the way a real production environment would require.

2. Objectives
•Build a custom VPC with fully isolated public and private subnet tiers across two Availability Zones.
•Provide private resources with outbound internet access without exposing them to inbound traffic from the internet.
•Enforce access control using security-group references instead of hardcoded CIDR blocks, for a more scalable and auditable design.
•Validate the design empirically not just by inspecting configuration, but by proving connectivity and isolation behaviorally.

3. Architecture
The VPC uses a /16 address space, split into four /24 subnets across two Availability Zones two public-facing and two private. Only the public subnets have a route to the Internet Gateway; private subnets route outbound traffic through a NAT Gateway instead.

Network Layout
Resource	CIDR / Detail	Availability Zone
VPC (secure-vpc-project)	10.0.0.0/16	—
public-subnet-1	10.0.1.0/24	us-east-1a
public-subnet-2	10.0.2.0/24	us-east-1b
private-subnet-1	10.0.11.0/24	us-east-1a
private-subnet-2	10.0.12.0/24	us-east-1b

Core Components
•Internet Gateway (secure-vpc-igw) attached to the VPC, provides the public subnets' route to the internet.
•NAT Gateway (secure-vpc-nat) deployed in public-subnet-1 with an automatically-assigned Elastic IP; gives private subnets outbound-only internet access.
•Route Tables public-rt (0.0.0.0/0 → IGW) associated with both public subnets; private-rt (0.0.0.0/0 → NAT Gateway) associated with both private subnets.
•Security Groups  public-sg and private-sg (detailed in Section 4).

<img width="1210" height="696" alt="image" src="https://github.com/user-attachments/assets/772b52ca-696f-4a35-a23e-3814d51900db" />


High-Level Design -- VPC across two Availability Zones; single NAT Gateway in AZ-1 (with Elastic IP) provides outbound access for both private subnets, including a cross-AZ dependency for private-subnet-2

Design Trade-off: Single NAT Gateway
The NAT Gateway is deployed in only one Availability Zone (us-east-1a) rather than one per AZ. This is a deliberate cost-saving decision for a portfolio/demo build a production environment would typically deploy one NAT Gateway per AZ to avoid a single point of failure for outbound connectivity, at roughly double the hourly cost.

3. Networking Configuration
VPC & Subnets
The VPC (secure-vpc-project) was created with a 10.0.0.0/16 CIDR block, no IPv6, and default tenancy. Four subnets were carved out of this range as shown in Section 2, split evenly across two Availability Zones to support a highly-available design pattern even though this project's NAT Gateway itself is single-AZ.

<img width="1120" height="620" alt="image" src="https://github.com/user-attachments/assets/d481e31c-5cf5-40ff-9bbe-6af2e8b1d49a" />

Figure 1 -- secure-vpc-project VPC details in the AWS Console

<img width="1120" height="622" alt="image" src="https://github.com/user-attachments/assets/9e8e1285-0089-47ec-bda3-9a62e0ce17a6" />


Figure 2 -- All four subnets (public-subnet-1/2, private-subnet-1/2) listed as Available

Internet Gateway
secure-vpc-igw was created and attached to the VPC. Without this attachment, no traffic public or private can reach or be reached from the internet, regardless of subnet or route table configuration.

<img width="1120" height="624" alt="image" src="https://github.com/user-attachments/assets/c369ec4f-c68b-40eb-a35d-ad98c3e84ef5" />

Figure 3 -- secure-vpc-igw shown as Attached to secure-vpc-project

NAT Gateway
secure-vpc-nat was created inside public-subnet-1, using Public connectivity type and automatic Elastic IP allocation. This means private-subnet resources initiate outbound connections that appear to originate from the NAT Gateway's Elastic IP the private instances themselves are never directly addressable from the internet.

<img width="1120" height="622" alt="image" src="https://github.com/user-attachments/assets/5beccca7-99ba-4bc6-8d86-9c1110f364a2" />


Figure 4 -- secure-vpc-nat shown as Available, Public connectivity, Regional

<img width="1120" height="624" alt="image" src="https://github.com/user-attachments/assets/79cb7783-3063-43fd-b493-5655c07fdf7f" />

Figure 5 -- Elastic IP allocated and associated with the NAT Gateway

Route Tables
Route Table	Destination	Target
public-rt	10.0.0.0/16	local
public-rt	0.0.0.0/0	secure-vpc-igw (Internet Gateway)
private-rt	10.0.0.0/16	local
private-rt	0.0.0.0/0	secure-vpc-nat (NAT Gateway)

public-rt is associated with public-subnet-1 and public-subnet-2. private-rt is associated with private-subnet-1 and private-subnet-2. This is the step most prone to misconfiguration pointing a private subnet's default route at the Internet Gateway instead of the NAT Gateway would inadvertently expose it.

<img width="1120" height="604" alt="image" src="https://github.com/user-attachments/assets/6bc48eb6-c1cf-411c-967e-f173c2b3fac3" />

Figure 6 -- public-rt associated with 2 subnets, routing 0.0.0.0/0 to the Internet Gateway

<img width="1120" height="622" alt="image" src="https://github.com/user-attachments/assets/d0631ff7-0838-4240-8715-fe755fd660be" />


Figure 7 -- private-rt associated with 2 subnets, routing 0.0.0.0/0 to the NAT Gateway

4. Security Groups
Two security groups were created to enforce a strict boundary between the public and private tiers.
public-sg
Type	Protocol / Port	Source
HTTP	TCP / 80	0.0.0.0/0
HTTPS	TCP / 443	0.0.0.0/0
SSH	TCP / 22	My IP only

private-sg
Type	Protocol / Port	Source
SSH (and app traffic as needed)	TCP / 22	public-sg (security group reference)

<img width="1120" height="624" alt="image" src="https://github.com/user-attachments/assets/41b601a5-0ed8-47b8-838c-72ca534d39d8" />

Figure 8 -- public-sg created successfully: allows HTTP/HTTPS from internet, SSH from admin IP only

<img width="1120" height="622" alt="image" src="https://github.com/user-attachments/assets/50ab4bd9-8fe7-4f8d-ba5a-f95085c51f8f" />

Figure 9 -- private-sg created successfully: allows traffic only from public-sg, no direct internet exposure

Design Decision: Security-Group Referencing Over CIDR Rules
private-sg's inbound rule references public-sg directly as its source, rather than allowing a CIDR range such as 10.0.1.0/24. This means any instance carrying the public-sg security group is automatically trusted, regardless of its specific IP address including if instances are replaced, or if an Auto Scaling Group launches new ones. This avoids hard coded IP maintenance and is the pattern used in the three-tier AWS project's security group design as well, reflecting a consistent, production-style approach across both portfolio builds.

5. Testing & Validation
Two EC2 instances (Amazon Linux 2023, t2.micro) were launched to validate the design empirically:
•public-test-instance — launched in public-subnet-1, with an auto-assigned public IP and public-sg attached.
•private-test-instance — launched in private-subnet-1, with no public IP and private-sg attached.

<img width="1120" height="628" alt="image" src="https://github.com/user-attachments/assets/1611085e-ff73-44d8-99cf-fe3d72e62e5b" />

Figure 10 -- public-test-instance running with public IPv4 address 3.239.19.236 and private IP 10.0.1.28

<img width="1120" height="628" alt="image" src="https://github.com/user-attachments/assets/585c2404-f89d-491a-a735-ff2b0b8a10ca" />

Figure 11 -- private-test-instance running with private IP 10.0.11.80 and no public IPv4 address

Connection Method: SSH Agent Forwarding
Rather than copying the private key (.pem file) onto the public instance to enable the onward hop to the private instance, SSH agent forwarding was used instead. This keeps the private key on the local machine at all times the public instance never has direct access to the key material, which is the more secure approach for a bastion-style hop.
ssh-add C:\Users\kubra\Downloads\secure-vpc-key.pem
ssh -A -i C:\Users\kubra\Downloads\secure-vpc-key.pem ec2-user@<public-instance-public-IP>

# From inside the public instance, no key file present:
ssh ec2-user@<private-instance-private-IP>
Evidence

<img width="1000" height="576" alt="image" src="https://github.com/user-attachments/assets/7be1ab42-eed1-4895-a8d6-6d458f1f9c2e" />

Figure 12 -- Successful SSH connection into public-test-instance (Amazon Linux 2023 banner)

<img width="1120" height="766" alt="image" src="https://github.com/user-attachments/assets/17cedba3-94da-4770-8f1f-0a5430d26f3b" />

Figure 13 -- Successful SSH hop from public-test-instance into private-test-instance via agent forwarding
<img width="1120" height="330" alt="image" src="https://github.com/user-attachments/assets/2d577f13-ad80-47e9-9caa-7aba15f6d271" />

Figure 14 -- curl from private-test-instance returns HTTP/2 200 via the NAT Gateway, confirming outbound internet access

Test Results
Test	Expected Result	Actual Result
SSH into public-test-instance from local machine	Connects successfully	Passed — connected, Amazon Linux 2023 banner shown
SSH hop from public-test-instance to private-test-instance	Connects using forwarded key, no .pem present on public instance	Passed — private-sg correctly trusted public-sg as source
Outbound internet access from private-test-instance	curl to an external HTTPS endpoint returns a valid response	Passed — HTTP/2 200 returned from https://www.google.com via NAT Gateway

Note on Isolation Testing
A direct connection attempt from the local machine straight to the private instance's private IP (bypassing the public instance entirely) was identified as a valuable additional test to prove inbound isolation, since private IP addresses are not routable from outside the VPC in any case, and private-sg does not permit any source other than public-sg.
7. Lessons Learned
•Route table misconfiguration is the most common failure point — a private subnet accidentally routed to the Internet Gateway instead of the NAT Gateway silently defeats the entire isolation design.
•SSH agent forwarding is a materially better practice than copying private key files onto intermediate (bastion) instances, since it never exposes key material outside the local machine.
•Security-group-to-security-group references scale better than CIDR-based rules, since they remain correct even if instances are replaced or IP ranges change.
•A single-AZ NAT Gateway is an acceptable and common cost trade-off for demo/portfolio environments, but should be called out explicitly rather than presented as a production-grade HA design.


