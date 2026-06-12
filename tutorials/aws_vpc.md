# Virtual Private Cloud (VPC)

## VPC with one public and one private subnet

![publicVPC][networking_project_vpc1]

AWS VPC (Virtual Private Cloud) is a service that allows users to create a logically isolated virtual network within the AWS cloud infrastructure. 
Users can define their own IP address range, subnets, and network gateways, and configure security rules to control access to their resources.

> [!IMPORTANT]
> Creating the VPC itself does not incur any cost. You'll be charged only when launching a virtual machines within your VPC. 

### Create a VPC

1. Open the Amazon VPC console at [https://console\.aws\.amazon\.com/vpc/](https://console.aws.amazon.com/vpc).

2. In the navigation pane, choose **Your VPCs**, **Create VPC**\.

3. Under **Resources to create**, choose **VPC only**\.

4. Specify the following VPC details\.
    + **Name tag**: Provide a name for your VPC, e.g. `<my-alias>-vpc` (change `<my-alias>` to your alias). 
    + **IPv4 CIDR block**: Specify an IPv4 CIDR block of `10.0.0.0/16` for your VPC\. The CIDR block size must have a size between /16 and /28\. More information can be found in [RFC 1918](http://www.faqs.org/rfcs/rfc1918.html).
    + **Tenancy**: Choose the `default` tenancy option for this VPC\.

5. Choose **Create VPC**\.

Great! Enter the main page of your VPC and explore the details. Especially, note the DHCP option set, the main route table, and the network ACL that AWS was allocated for your VPC. 

**DHCP option set**

No any good reason to touch it, AWS done all for you!

**Network ACL**

A network access control list (ACL) is an optional layer of security for your VPC that acts as a firewall for controlling traffic in and out of one or more subnets.

**Main route table**

The main route table contains a single rule that allows traffic to flow freely within the VPC, but does not allow traffic to flow in or out of the VPC.
The main route table is applied by default to all subnets in the VPC that are not explicitly associated with a different route table.
Thus, by default, machines that will be part of the VPC would not be able to access the public internet, unless otherwise defined. 

### Create the public subnet in your VPC

To add a new subnet to your VPC, you must specify an IPv4 CIDR block for the subnet from the range of your VPC\.
You can specify the Availability Zone in which you want the subnet to reside\.

1. In the navigation pane, choose **Subnets**\.

2. Choose **Create subnet**\.

3. For **VPC ID**: Choose the VPC for the subnet\.

4. For **Subnet name**, enter a name for your subnet: `<my-alias>-public-subnet`. 

5. For **Availability Zone**, you can choose a Zone for your subnet, or leave the default **No Preference** to let AWS choose one for you\.

6. For **IPv4 CIDR block**, enter an IPv4 CIDR block for your subnet\: `10.0.0.0/24`\.

7. Choose **Create subnet**\.

### Create a custom route table

To allow traffic to flow out of the VPC (to the public internet), a new custom route table needs to be created and associated with the above created subnet.

We will turn now to create the custom route table that would allow traffic flow to the public internet. 

1. In the navigation pane, choose **Route Tables**\.

2. Choose **Create route table**\.

3. For **Name**, enter a name for your route table. E.g. `<my-alias>-public-rt`.

4. For **VPC**, choose your VPC\.

5. Choose **Create**\.

### Create and attach an Internet Gateway

In a real, physical network, the way computers are able to connect to the internet is by a device called **a router**. But since we are building virtual network, the way of AWS to provide us a "router", is by a resource called **Internet Gateway**.
We can say that **Internet Gateway** serves as a virtual router to connect your VPC to the internet.

You'll now create an internet gateway and attach it to your VPC.

1. In the navigation pane, choose **Internet Gateways**, and then choose **Create internet gateway**\.

2. Name your internet gateway, e.g. `<my-alias>-igw`.

3. Choose **Create internet gateway**\.

4. Select the internet gateway that you just created, and then choose **Actions, Attach to VPC**\.

5. Select your VPC from the list, and then choose **Attach internet gateway**\.

### Add the Internet Gateway as a target in a custom route table

When you create a subnet, AWS automatically associate it with the main route table for the VPC\.
By default, the main route table doesn't contain a route to an internet gateway\.
The following procedure uses your custom route table and creates a route that sends traffic destined outside the VPC to the internet gateway, and then associates it with your subnet\.

1. Select the custom route table that you just created\. The details pane displays tabs for working with its routes, associations, and route propagation\.

2. On the **Routes** tab, choose **Edit routes**, **Add route**, and add the following routes as necessary\. Choose **Save changes** when you're done\.
    + For IPv4 traffic, specify `0.0.0.0/0` in the **Destination** box, and select the internet gateway ID in the **Target** list\.

3. On the **Subnet associations** tab, choose **Edit subnet associations**, select the check box for the subnet, and then choose **Save associations**\.

### Auto assign public IP for every machine that will be connected to the public subnet

Each subnet has an attribute that determines whether machines launched into that subnet are assigned a public IP address by default.
Your created subnet has this attribute set to false. We would like to enable this property, so we would be able to connect to machines in this subnet from our local machine. 

1. In the navigation pane, choose **Subnets**\.

2. Select the public subnet that you just created, and then choose **Actions, Edit subnet settings**\.

3. For **Auto-assign IP settings**, check the **Enable auto-assign public IPv4 address** box. 

4. Choose **Save**\.





[networking_project_stop]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_project_stop.gif
[networking_project_vpc1]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/networking_project_vpc1.png
[networking_project]: https://github.com/exit-zero-academy/NetworkingProject
