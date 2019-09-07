# How To: IPv6 VPN to AWS

A guide for building an IPv6 site-to-site VPN to AWS



-----------------------------------------
As of the creation of this document (September of 2019), there is no native support in the AWS site-to-site VPN capabilities for IPv6 as either the underlay protocol or as the tunneled protocol. This can be troublesome especially since there is good support for IPv6 in AWS VPCs for EC2 instances.

This document provides a workaround solution for this limitation by using an **EC2 Ubuntu instance** enabled with the **strongSwan IPSEC packages** to terminate an IPv6 VPN tunnel between an AWS VPC and a remote VPN concentrator. The Cisco ASA platform is used in the examples as the remote or 'right' side VPN concentrator.



-----------------------------------------
## Overview
- **Example Settings**
- **Launch a VPN Gateway EC2 instance**
- **Configure your VPC for new VPN**
- **Configure the remote VPN concentrator**
- **Install and configure strongSwan**
- **OPTIONAL: IPv4 and IPv6 over the New Tunnel**



-----------------------------------------
## Example Settings
Below are the details used in the examples in this document

| Name                       | Value                                               |
|----------------------------|-----------------------------------------------------|
| AWS EC2 AMI                | **[Ubuntu Linux 18.04 LTS Bionic](https://aws.amazon.com/marketplace/pp/B07CQ33QKV)**  |
| EC2 Instance Size          | **t3a.nano**                                        |
| AWS-Side Peer IP           | **2001:DB8:0:A::10**                                |
| Customer Peer IP           | **2001:DB8:0:C::10**                                |
| AWS-Side Tunneled Network  | **2001:DB8:A::/48**                                 |
| Customer Tunneled Network  | **2001:DB8:C::/48**                                 |
| IPSEC Pre-Shared Key       | **somesupersecretkey**                              |



-----------------------------------------
## Launch a VPN Gateway EC2 instance
The particular Linux distribution/version and the instance type you use for this can vary. The Ubuntu t3a.nano instance used in this example has a cost of **$0.0047 per hour**

Since this instance will need to be accessible from the internet for the underlay (with either IPv4 or IPv6), make sure you are using an Internet Gateway for the default route for the instance's subnet (you should see something similar to `::/0       igw-1a2b3c4d` or `0.0.0.0/0       igw-1a2b3c4d` in the route table).

The purpose of all this is to make sure a host on the internet can properly reach the EC2 instance on the IPSEC ports using its public IPv6 or IPv4 address

Make sure to set up and attach a security group to your instance's interface so you can reach it via SSH using its public IPv4 or IPv6. You will also need to allow the appropriate IPSEC ports through the security group, although I just allow all protocols from the specific public peer IP of the remote side.

For the example, make sure to assign the IPv6 address **2001:DB8:0:A::10** to the Ubuntu instance



-----------------------------------------
## Configure your VPC for new VPN

- Adjust the network settings on the instance by disabling source address checking
  - This needs to be done to allow the instance to de-encapsulate VPN packets and let them emerge onto the VPC from the instance's interface
  - This can be found by selecting the EC2 instance and going to **Actions > Networking > Change Source/Dest. Check**

- Adjust the route tables for your subnets to route the customer tunneled /48 network towards the instance, but route the customer VPN concentrator peer IP towards the Internet Gateway for the subnet holding the Ubuntu instance
  - This would look something like the Below

```
2001:DB8:C::/48              eni-1a2b3c4d5e6f
2001:DB8:0:C::10/128         igw-1a2b3c4d
```

- If you are already routing `::/0` towards an Internet Gateway and the Customer Tunneled Network does not overlap with the Customer Peer IP, then the /128 route is not necessary



-----------------------------------------
## Configure the remote VPN concentrator

Use the below configuration as a template for your ASA configuration

```
crypto ipsec ikev1 transform-set AES_SHA_HMAC esp-aes esp-sha-hmac
!
crypto ikev1 policy 10
 authentication pre-share
 encryption aes
 hash sha
 group 2
 lifetime 28800
!
access-list ACL_AWS_IPV6_VPN extended permit ip 2001:DB8:C::/48 2001:DB8:A::/48
!
tunnel-group 2001:DB8:0:A::10 type ipsec-l2l
tunnel-group 2001:DB8:0:A::10 ipsec-attributes
 ikev1 pre-shared-key somesupersecretkey
!
crypto map VPN_MAP 10 match address ACL_AWS_IPV6_VPN
crypto map VPN_MAP 10 set pfs group2
crypto map VPN_MAP 10 set peer 2001:DB8:0:A::10
crypto map VPN_MAP 10 set ikev1 transform-set AES_SHA_HMAC
crypto map VPN_MAP 10 set security-association lifetime seconds 3600
!
crypto ikev1 enable OUTSIDE
crypto map OUTSIDE_MAP interface OUTSIDE
```



-----------------------------------------
## Install and configure strongSwan
Once the Linux instance is up and running and accessible via SSH, we need to install and configure strongSwan

- Install the strongSwan package with the command `sudo apt install strongswan -y`
- Modify the IPv6 forwarding functionality on the server in the `/etc/sysctl.conf` file with `sudo vi /etc/sysctl.conf`
  - Uncomment the below line
```
net.ipv6.conf.all.forwarding=1
```
- Reboot the server after this change to have it take effect
- Set the IPSEC PSK in `/etc/ipsec.secrets` with `sudo vi /etc/ipsec.secrets`
  - Add the below line to the file
```
    2001:DB8:0:C::10 : PSK "somesupersecretkey"
```
- Set the VPN connection settings in `/etc/ipsec.conf` with `sudo vi /etc/ipsec.conf`
  - Add the below config to the bottom of the file

```
conn %default
        keyexchange=ikev1


conn customer_vpn_v6
        left=2001:DB8:0:A::10
        right=2001:DB8:0:C::10
        leftsubnet=2001:DB8:A::/48
        rightsubnet=2001:DB8:C::/48
        ike=aes128-sha1-modp1024!
        keyexchange=ikev1
        ikelifetime=28800s
        esp=aes128-sha1-modp1024!
        keylife=3600s
        rekeymargin=540s
        type=tunnel
        compress=no
        authby=secret
        auto=start
        keyingtries=%forever
```

- Restart the IPSEC service with `sudo ipsec restart`
- Check the VPN status with `sudo ipsec status`
- Once the VPN comes up, you should be able to test end-to-end IPv6 reachability



-----------------------------------------
## OPTIONAL: IPv4 and IPv6 over the New Tunnel
If you have made it this far, you may be asking yourself why you would want to run IPv6 over a strongSwan IPSEC tunnel on an EC2 instance, and IPv4 over a managed AWS VPN service. There are a couple of benefits which come to mind with the AWS service

1. **Redundancy**: AWS gives you multiple peer IPs to use for the managed VPN service which provides a level of redundancy within a region. They also allow you to automatically propagate the VPN tunneled routes into the VPC Route Tables when the VPN comes up to further enhance this functionality

2. **Ease of Management**: The managed VPN service is managed by AWS and does not require any kind of regular updates or the typical care and feeding of a Linux instance

Those benefits aside, there are a few benefits to abandoning the managed VPN service and using the EC2 instance for both IPv4 and IPv6:

1. **Cost**: The cost of the managed VPN service from AWS in the Ohio region is **$0.05 per hour** which equates to about **$36 per month**. The cost of a t3a.nano instance running strongSwan is **$0.0047 per hour** which equates to about **$3.38 per month**: about one-tenth the cost

2. **Functionality**: If you are a bit network-savvy, then you likely know that you can take a solution like this to a whole different level if desired. Using a VTI with routing protocols, multiple tunnels with multiple paths, the sky is the limit when it comes to networking using Linux. Running IPv4 + IPv6 over the same VPN tunnel is only the beginning



### Tunneled IPv4 Example Settings
Below are the details used in the examples in this document

| Name                       | Value                                               |
|----------------------------|-----------------------------------------------------|
| AWS-Side Tunneled Network  | **172.31.0.0/20**                                   |
| Customer Tunneled Network  | **192.168.0.0/24**                                  |



Below are the steps to extend the IPv6 tunnel built above to also carry IPv4

- Modify the IPv4 forwarding functionality on the server in the `/etc/sysctl.conf` file with `sudo vi /etc/sysctl.conf`
  - Uncomment the below line
```
net.ipv4.ip_forward=1
```

- Reboot the server after this change to have it take effect

- Delete the current AWS Site-to-Site VPN, Customer Gateway, and Virtual Private Gateway if they exist

- Set routes in your route-tables to point at the EC2 instance for your remote tunneled networks

- Make some modifications on the ASA VPN ACL to include both tunneled address-families as below

### Modified ASA Crypto Config

```
access-list ACL_AWS_IPV6_VPN extended permit ip 2001:DB8:C::/48 2001:DB8:A::/48
access-list ACL_AWS_IPV6_VPN extended permit ip 192.168.0.0 255.255.255.0 172.31.0.0 255.255.240.0
```

- Since we are now dealing with the legacy protocol (IPv4) which often uses NAT, you may need to add some NAT-exemption statments to the ASA for the VPN tunnel
  - If you already have these NAT statements on the firewall (since you were running an IPv4 VPN to AWS previously), then you likely don’t need to change them

- Delete your old IPv4 crypto-map entry pointed at AWS since you will now be using the same tunnel for that old IPv4 space as well as your shiny new IPv6 space

- Modify the `/etc/ipsec.conf` settings on the Ubuntu instance with `sudo vi /etc/ipsec.conf` to reflect the below


## FILE: /etc/ipsec.conf

```
conn %default
        keyexchange=ikev1


conn customer_vpn_v6
        left=2001:DB8:0:A::10
        right=2001:DB8:0:C::10
        leftsubnet=2001:DB8:A::/48
        rightsubnet=2001:DB8:C::/48
        ike=aes128-sha1-modp1024!
        keyexchange=ikev1
        ikelifetime=28800s
        esp=aes128-sha1-modp1024!
        keylife=3600s
        rekeymargin=540s
        type=tunnel
        compress=no
        authby=secret
        auto=start
        keyingtries=%forever

conn customer_vpn_v4
        left=2001:DB8:0:A::10
        right=2001:DB8:0:C::10
        leftsubnet=172.31.0.0/20
        rightsubnet=192.168.0.0/24
        ike=aes128-sha1-modp1024!
        keyexchange=ikev1
        ikelifetime=28800s
        esp=aes128-sha1-modp1024!
        keylife=3600s
        rekeymargin=540s
        type=tunnel
        compress=no
        authby=secret
        auto=start
        keyingtries=%forever
```

- Restart the IPSEC service with `sudo ipsec restart`
- Check the VPN status with `sudo ipsec status`
- Once the VPN comes up, you should be able to test end-to-end IPv6 reachability
