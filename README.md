# My AWS Networking Project: Building a Scalable VPC Infrastructure from Scratch

**Author:** Prashant Gohel
**Date:** September 9, 2025

---

### **Table of Contents**
1. [My Project's Goal](#1-my-projects-goal)
2. [Core Concepts: The "What & Why"](#2-core-concepts-the-what--why)
3. [The Step-by-Step Guide I Followed](#3-the-step-by-step-guide-i-followed)
    * [Part 1: Building My First VPC ('test-vpc')](#part-1-building-my-first-vpc-test-vpc)
    * [Part 2: Creating a Second, Isolated VPC ('production-vpc')](#part-2-creating-a-second-isolated-vpc-production-vpc)
    * [Part 3: Connecting VPCs with Peering](#part-3-connecting-vpcs-with-peering)
    * [Part 4: Creating a Secure Private Subnet with a NAT Gateway](#part-4-creating-a-secure-private-subnet-with-a-nat-gateway)
4. [Critical Final Step: Cleaning Up My AWS Resources](#4-critical-final-step-cleaning-up-my-aws-resources)
5. [Final Thoughts and Conclusion](#5-final-thoughts-and-conclusion)

---

### **1. My Project's Goal**
<a name="1-my-projects-goal"></a>
For this project, I decided to master the foundational concepts of networking on AWS by building a complete cloud infrastructure from the ground up. My goal was to move beyond single-server deployments and understand how to create secure, isolated environments and enable private communication between them.

The plan was to:
1.  Create two completely separate Virtual Private Clouds (VPCs) to simulate `test` and `production` environments.
2.  Launch virtual servers (EC2 instances) within these VPCs.
3.  Securely provide internet access to public-facing servers.
4.  Establish a private, secure connection between the two VPCs using VPC Peering.
5.  Create a secure private subnet for backend services (like databases) that could initiate outbound internet connections for updates without being exposed to inbound traffic.

Finally, a crucial part of the project was learning how to decommission all the resources in the correct order to ensure my AWS account remained clean and free of unexpected charges.

---

![VPC Infrastructure Diagram](Diagram.gif)

### **2. Core Concepts: The "What & Why"**
<a name="2-core-concepts-the-what--why"></a>
Before diving into the steps, I wanted to solidify my understanding of the key components I would be working with.

* **What is a VPC (Virtual Private Cloud)?** I think of a VPC as my own private, isolated section of the AWS cloud. It's like buying a plot of land. By default, anything I build inside my VPC is completely cut off from the rest of the world and even other VPCs, giving me full control over security and access.

* **What is a Subnet?** If a VPC is my plot of land, a Subnet is a room inside my house. I can designate some rooms as "public" (like a living room) and others as "private" (like a bedroom). In AWS terms, a **public subnet** is one that can have a direct route to the internet, while a **private subnet** does not.

* **What is an Internet Gateway (IGW)?** This is the single, secure doorway for my entire house. An IGW is a component that I attach to my VPC to allow two-way communication with the internet. Without it, my VPC is a completely closed box.

* **What is a Route Table?** A Route Table is the GPS or internal signage system for my VPC. It contains a set of rules, called routes, that determine where network traffic from my subnets is directed. For a subnet to be "public," its route table must have a route (`0.0.0.0/0`) that points all internet-bound traffic to the Internet Gateway.

* **What is a NAT Gateway?** A NAT (Network Address Translation) Gateway is a managed AWS service that solves a critical security problem: how to let servers in a private subnet (like a database) download software updates from the internet without exposing them to inbound connections. The NAT Gateway lives in the public subnet and acts like a one-way door, allowing private instances to go out to the internet while blocking any unsolicited traffic from coming in.

---

### **3. The Step-by-Step Guide I Followed**
<a name="3-the-step-by-step-guide-i-followed"></a>

#### **Part 1: Building My First VPC ('test-vpc')**
My first step was to create the `test` environment.

1.  **VPC Creation:** In the AWS VPC dashboard, I chose to create a "VPC only" to manually configure each component.
    * **Name:** `test-vpc`
    * **IPv4 CIDR:** `10.0.0.0/16`. I chose this because it's a standard private IP range and the `/16` gives me a massive pool of 65,536 IPs, which is more than enough for any project.
2.  **Subnet Creation:** I created a public subnet within my VPC.
    * **Name:** `test-public-subnet`
    * **CIDR:** `10.0.0.0/24`. I carved out a smaller `/24` block (256 IPs) from my VPC's range for this specific subnet.
3.  **Internet Gateway (IGW):** I created an IGW named `test-igw` and, crucially, **attached** it to my `test-vpc`.
4.  **Route Table:** I created a route table named `test-public-rt`.
    * I first associated it with my `test-public-subnet`.
    * Then, I edited its routes and added the most important rule for public access: **Destination:** `0.0.0.0/0` (anywhere on the internet) -> **Target:** my `test-igw`.
5.  **EC2 Instance Launch:** I launched a `t2.micro` Ubuntu instance, making sure to place it inside my `test-vpc` and `test-public-subnet`. The most critical setting was to **Enable** "Auto-assign Public IP." Without this and the IGW/Route Table setup, I would have no way to SSH into my server from the internet.

#### **Part 2: Creating a Second, Isolated VPC ('production-vpc')**
I repeated the exact same process to create my `production` environment, but with different names and, most importantly, a different IP range to prevent any overlap.
* **VPC Name:** `production-vpc` with CIDR `192.168.0.0/16`
* **Subnet Name:** `production-public-subnet` with CIDR `192.168.0.0/24`

At this point, I had two EC2 instances in two completely separate VPCs that had no knowledge of each other.

#### **Part 3: Connecting VPCs with Peering**
<a name="part-3-connecting-vpcs-with-peering"></a>
My goal here was to allow the test instance to communicate with the production instance using their private IP addresses, simulating a secure backend connection.

1.  **Create Peering Connection:** In the VPC dashboard, I created a peering connection, naming it `test-to-prod-peering`. I set `test-vpc` as the "requester" and `production-vpc` as the "accepter."
2.  **Accept the Request:** I then had to find the pending request and formally accept it to activate the connection.
3.  **Update Route Tables (Crucial Step):** The connection doesn't work until you tell the VPCs how to use it.
    * In the `test-public-rt`, I added a route: **Destination:** `192.168.0.0/16` (the entire Production VPC) -> **Target:** the Peering Connection.
    * In the `production-public-rt`, I added the reverse route: **Destination:** `10.0.0.0/16` (the entire Test VPC) -> **Target:** the same Peering Connection.
4.  **Update Security Groups:** The final step was to update the instance firewalls (Security Groups). I allowed `All ICMP` (ping) traffic, but instead of allowing it from anywhere, I set the **Source** to be the CIDR block of the *other* VPC. This ensures that only instances within my peered VPC can communicate, not the entire internet.

After this, I successfully pinged the private IP of my production instance from my test instance.

#### **Part 4: Creating a Secure Private Subnet with a NAT Gateway**
<a name="part-4-creating-a-secure-private-subnet-with-a-nat-gateway"></a>
This was the final piece of the infrastructure, designed for backend servers.

1.  **Private Subnet Creation:** I created a new subnet in my `test-vpc` named `test-private-subnet` with a non-overlapping CIDR of `10.0.1.0/24`.
2.  **NAT Gateway Creation:** I created a NAT Gateway named `test-nat-gw`. Critically, I placed it **inside my public subnet** (`test-public-subnet`) and allocated a new Elastic IP for it, as it needs a public IP to function.
3.  **Private Route Table:** I created a new route table, `test-private-rt`, and associated it with my `test-private-subnet`.
4.  **Configure Private Routing:** The key to making this work was the route I added to `test-private-rt`: **Destination:** `0.0.0.0/0` -> **Target:** my newly created **NAT Gateway**.

This setup means any instance launched in my private subnet (without a public IP) will route its internet-bound traffic through the NAT Gateway, allowing it to download updates securely.

---

### **4. Critical Final Step: Cleaning Up My AWS Resources**
<a name="4-critical-final-step-cleaning-up-my-aws-resources"></a>
To avoid any charges, I learned that I must delete resources in a specific order, because of their dependencies.
1.  **EC2 Instances:** Terminate all instances first.
2.  **NAT Gateway:** This must be deleted before the VPC. Deleting it also releases its costly Elastic IP.
3.  **Peering Connection, IGWs, Subnets, and Route Tables:** I deleted all the networking components.
4.  **VPCs:** Finally, with everything inside them gone, I could delete the VPCs themselves.

---

### **5. Final Thoughts and Conclusion**
<a name="5-final-thoughts-and-conclusion"></a>
This project was an incredible deep dive into AWS networking. By building everything manually, I moved beyond theory and gained a practical understanding of how to design and implement a secure, scalable, and interconnected cloud environment. Mastering these concepts is essential for any cloud or DevOps role, and this hands-on experience has been invaluable.
  