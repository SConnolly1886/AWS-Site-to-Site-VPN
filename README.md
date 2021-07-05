
# AWS Site-to-Site VPN with Dynamic Routing

This will be a walkthrough on how how to set up a site-to-site VPN with AWS

### What will be created

The AWS environment will have the following:
- 2 subnets
- 2 EC2 instances
- TGW
- default route pointing to TGW

A simulated on-prem environment will have the following:
- 1 public subnet
- 2 private subnets
- 2 ubuntu routers with Strongswan (Router1 and Router2) that you will connect through via VPN endpoints

![image](https://user-images.githubusercontent.com/62077185/124508816-9936aa80-dd9e-11eb-9c68-8675b55b5402.png)


## Setting up the environments

Use the cloudformation templates to set up the respective environments<br />
-S2SVPN-AWS<br />
-S2SVPN-ON-P<br />

As usual wait out the setup...

## Router IPs

When setting up a site-to-site VPN you must make note of the IP address for the on-prem side of things. You will need the IP for the Router(s) as well as the internal CIDR that you intend to connect via VPN. 

In this example the routers IP's can be found as outputs: `Router1Public` and `Router2Public`

## Creating On-prem Customer Gateways (CGW)
In this example there are 2 on-premises routers therefore 2 customer gateways are required.

### How do you set up a CGW for Router1?
-Select `Customer Gateways` under `Virtual private Network (VPN)` and create a CGW
-Set the name of the CGW as `ROUTER1` and click `Dynamic` for routing  
-Set BGP ASN to `65016` 
-Set IP Address to Router1PubIP (The public IP for Router1)  
-create `Customer gateway`  

![selectcustomergateway](https://user-images.githubusercontent.com/62077185/124516464-b2942280-ddaf-11eb-9809-9ba72aaffa1e.JPG)

** Note: You can set a private BGP ASN in the range of `64512 to 65535` ** 
  
### How do you set up a CGW for Router2?
That's right, it's the same as Router1. So just repeat the steps:
-Select `Customer Gateways` under `Virtual private Network (VPN)`  and create a CGW
-Set the name of the CGW as `ROUTER2` and click `Dynamic` for routing  
-Set BGP ASN to `65016` (the same as Router1)
-Set IP Address to Router2PubIP (The public IP for Router2)     
-`create Customer gateway`  

## Is there connectivity?
Select `ONPREM-SERVER2` EC2 instance and connect to it through SSM Session Manager and try pinging EC2-B `ping IP_ADDRESS_OF_EC2-B`  
It won't work yet...there's still more to do

## Next step
Now 2 vpn attachments need to be created for the Transit Gateway (TGW). In essence its creating 2 vpn connections, 1 for each of the customer gateways in the on-prem environment. Each vpn connection has 2 tunnels for each AWS Endpoint pointing to the on-prem router.
This step is needed in order to download the config files (for configuring on-prem VPN endpoints)

### Create the VPN Attacments for TGW

In AWS, navigate to TGW attachments and create a TGW attacment. For transit gateway ID in the dropdown select the TGW created in the cloudformation stack.

![createtransattach](https://user-images.githubusercontent.com/62077185/124516484-bc1d8a80-ddaf-11eb-8824-ea7e36ce22f0.JPG)

-Attacment type: `VPN`
-`Customer gateway`: `Existing`
-`Customer gateway ID` : `ROUTER1`
-`Routing options` : `Dynamic`
-`Enable Acceleration`
-`Create Attachment`

`Create Transit Gateway Attachment`
`Transit Gateway ID` : `AWSTGW`
-Attacment type: `VPN`
-`Customer gateway`: `Ex-`Customer gateway ID` : `ROUTER2`
-`Routing options` : `Dynamic`
-`Enable Acceleration`
-`Create Attachment`

## Site-to-Site VPN
Move to `Site-to-Site VPN Connections` under `Virtual Private Network`

For each of the connections, it will show you the `Customer Gateway Address` these match `ROUTER1` and `ROUTER2`

-Select the line which matches Router1PubIP and `Download Configuration`
-Change vendor to `Generic` (Note select appropriate vendor if found, default to generic if need be)
-Download
-Rename this file to `Router1Config.TXT`
-Select the line which matches Router2PubIP
-`Download Configuration`
-Change vendor to `Generic`
-Download
-Rename this file to `Router2Config.TXT`

Great! Now the config files are available. This will allow us to set up the VPN Connection now.

## Configure the On-prem Ubuntu, Strongswan routers
Now the on-prem routers need to be configured so lets create IPSEC tunnels to the AWS environment. For each on-prem router you should create 2 IPSEC tunnels ... each going to a different AWS Endpoint. 

Make sure before starting this stage that both VPN connections are in an `available` state. Check your AWS console to verify.

### Configure the IPSEC Tunnels for Router1
Select the EC2 instance named `ROUTER1` and connect via SSM Session Manager

Once connected run the following commands:
- `sudo bash`  
- `cd /home/ubuntu/demo_assets/`  
- `vi ipsec.conf`  

This is is the file which configures the IPSEC Tunnel interfaces over which our VPN traffic flows.  
As we are connected to Router 1 - This configures the ones for ROUTER1 -> BOTH AWS Endpoints  

Replace the following placeholders with the real values found in the 2 config documents you downloaded for each router.

- `ROUTER1_PRIVATE_IP`  
- `ROUTER1_TUNNEL1_ONPREM_OUTSIDE_IP`  
- `ROUTER1_TUNNEL1_AWS_OUTSIDE_IP`  
- `ROUTER1_TUNNEL1_AWS_OUTSIDE_IP`  
and now for the second IPSEC tunnel...
- `ROUTER1_PRIVATE_IP`  
- `ROUTER1_TUNNEL2_ONPREM_OUTSIDE_IP`  
- `ROUTER1_TUNNEL2_AWS_OUTSIDE_IP`  
- `ROUTER1_TUNNEL2_AWS_OUTSIDE_IP`  

Save and exit

`vi ipsec.secrets`  

This file controls authentication for the tunnels  
Replace the following placeholders with the real values found in the config files

- `ROUTER1_TUNNEL1_ONPREM_OUTSIDE_IP`  
- `ROUTER1_TUNNEL1_AWS_OUTSIDE_IP`  
- `ROUTER1_TUNNEL1_PresharedKey`  
and  
- `ROUTER1_TUNNEL2_ONPREM_OUTSIDE_IP`  
- `ROUTER1_TUNNEL2_AWS_OUTSIDE_IP`  
- `ROUTER1_TUNNEL2_PresharedKey`  

Save and exit

`vi ipsec-vti.sh`  

This script brings UP the IPSEC tunnel interfaces when needed  
Replace the following placeholders with the real values in the config files

- `ROUTER1_TUNNEL1_ONPREM_INSIDE_IP`  (ensuring the /30 is at the end)  
- `ROUTER1_TUNNEL1_AWS_INSIDE_IP` (ensuring the /30 is at the end)  
- `ROUTER1_TUNNEL2_ONPREM_INSIDE_IP` (ensuring the /30 is at the end)  
- `ROUTER1_TUNNEL2_AWS_INSIDE_IP` (ensuring the /30 is at the end)  

Save and exit

`cp ipsec* /etc`  
`chmod +x /etc/ipsec-vti.sh`  

Now all the configuration for Router1 IPSEC has been completed, lets restart the strongSwan service to bring them up.  

`systemctl restart strongswan` to restart strongSwan ... this should bring up the tunnels  

We can check these tunnels are up by running  
`ifconfig`  
You should see `vti1` and `vti2` interfaces  
You can also check the connection in the AWS VPC Console ...the tunnels should be down, but IPSEC should be shown as UP after a few minutes.  


### Configure the IPSEC Tunnels for Router2
Select the EC2 instance named `ROUTER2` and connect via SSM Session Manager

Once connected run:
`sudo bash`  
`cd /home/ubuntu/demo_assets/`  
`nano ipsec.conf`  

This is is the file which configures the IPSEC Tunnel interfaces over which our VPN traffic flows.  
As we are connected to Router 2 - This configures the ones for ROUTER2 -> BOTH AWS Endpoints  

Replace the following placeholders with the real values in the config file 

- `ROUTER2_PRIVATE_IP`  
- `ROUTER2_TUNNEL1_ONPREM_OUTSIDE_IP`  
- `ROUTER2_TUNNEL1_AWS_OUTSIDE_IP`  
- `ROUTER2_TUNNEL1_AWS_OUTSIDE_IP`  
and  
- `ROUTER2_PRIVATE_IP`  
- `ROUTER2_TUNNEL2_ONPREM_OUTSIDE_IP`  
- `ROUTER2_TUNNEL2_AWS_OUTSIDE_IP`  
- `ROUTER2_TUNNEL2_AWS_OUTSIDE_IP`  

Save and exit

`vi ipsec.secrets`  

This file controls authentication for the tunnels  
Replace the following placeholders with the real values in the config file 

- `ROUTER2_TUNNEL1_ONPREM_OUTSIDE_IP`  
- `ROUTER2_TUNNEL1_AWS_OUTSIDE_IP`  
- `ROUTER2_TUNNEL1_PresharedKey`  
and  
- `ROUTER2_TUNNEL2_ONPREM_OUTSIDE_IP`  
- `ROUTER2_TUNNEL2_AWS_OUTSIDE_IP`  
- `ROUTER2_TUNNEL2_PresharedKey`  

Save and exit

`vi ipsec-vti.sh`  

This script brings UP the tunnel interfaces when needed  
Replace the following placeholders with the real values in the config file

- `ROUTER2_TUNNEL1_ONPREM_INSIDE_IP`  (ensuring the /30 is at the end)  
- `ROUTER2_TUNNEL1_AWS_INSIDE_IP` (ensuring the /30 is at the end)  
- `ROUTER2_TUNNEL2_ONPREM_INSIDE_IP` (ensuring the /30 is at the end)  
- `ROUTER2_TUNNEL2_AWS_INSIDE_IP` (ensuring the /30 is at the end)  

Save and exit

`cp ipsec* /etc`  
`chmod +x /etc/ipsec-vti.sh`  

Now all the configuration for Router1 IPSEC has been completed, lets restart the strongSwan service to bring them up.  

`systemctl restart strongswan` to restart strongSwan ... this should bring up the tunnels  

We can check these tunnels are up by running  
`ifconfig`  
You should see `vti1` and `vti2` interfaces  
You can also check the connection in the AWS VPC Console ...the tunnels should be down, but IPSEC should be shown as UP after a few minutes.  

### IPSEC check

When all 4 IPSEC TUNNELS are up (`vti1`,`vti2`,`Router1` and `Router2`) there still shouldn't be any connectivity. Next, lets add BGP capability with the dynamic BGP connections.

In this next step, lets use the IPSEC tunnels created earlier and add BGP sessions for all of the tunnels created.  
These sessions will allow the ONPREM Routers to exchange routers with the Transit Gateway running in AWS  
Once routes are exchanged, the connections will allow data to flow between AWS and ONPREMISES  
BGP capability is added using `FRR` and that will be installed on the on-prem routers.  

## Install FRR on Router1 (BGP capability)
Select the EC2 instance named `ROUTER1` and connect via SSM Session Manager

First we will make the `FRR` script executable and run it to install BGP capability.  
`sudo bash`  
`cd /home/ubuntu/demo_assets`   
`chmod +x ffrouting-install.sh`   
`./ffrouting-install.sh`  
** This will take some time - 10-15 minutes **  
** We can allow this to run, and start the same process on the other Router **  

## Install FRR on Router2 (BGP capability)
Select the EC2 instance named `ROUTER2` and connect via SSM Session Manager

`sudo bash`  
`cd /home/ubuntu/demo_assets`  
`chmod +x ffrouting-install.sh`     
`./ffrouting-install.sh`  

## Configure BGP routing for Router1

`vtysh`  
`conf t`  
`frr defaults traditional`  
`router bgp 65016`  
`neighbor ROUTER1_TUNNEL1_AWS_BGP_IP remote-as 64512`  
`neighbor ROUTER1_TUNNEL2_AWS_BGP_IP remote-as 64512`  
`no bgp ebgp-requires-policy`  
`address-family ipv4 unicast`  
`redistribute connected`  
`exit-address-family`  
`exit`  
`exit`  
`wr`  
`exit`  

`sudo reboot`  


`ROUTER1` once back will now be functioning as both an IPSEC endpoint and a BGP endpoint. It will be exchanging routes with the transit gateway in AWS.  

### Now test the connection
Select the EC2 instance named `ROUTER1` and connect via SSM Session Manager

`sudo bash`
`sudo chmod 740 /var/run/frr && systemctl restart frr`
SHOW THE ROUTES VIA THE UI `route`   
SHOW THE ROUTES VIA `vtysh`  
`show ip route`. 

Select the EC2 instance named `ONPREM-SERVER1` and connect via SSM Session Manager
run `ping IP_ADDRESS_OF_EC2-A`  

Select the EC2 instance named `EC2-A`  and connect via SSM Session Manager
run `ping IP_ADDRESS_OF_ONPREM-SERVER1`  


## Configure BGP routing for Router2

`vtysh`  
`conf t`  
`frr defaults traditional`  
`router bgp 65016`  
`neighbor ROUTER2_TUNNEL1_AWS_BGP_IP remote-as 64512`  
`neighbor ROUTER2_TUNNEL2_AWS_BGP_IP remote-as 64512`  
`no bgp ebgp-requires-policy`  
`address-family ipv4 unicast`  
`redistribute connected`  
`exit-address-family`  
`exit`  
`exit`  
`wr`  
`exit`  

`sudo reboot`  

`ROUTER2` once back will now be functioning as both an IPSEC endpoint and a BGP endpoint. It will be exchanging routes with the transit gateway in AWS.  
`EC2-A` 

Select the EC2 instance named `ROUTER1` and connect via SSM Session Manager

`sudo bash`     
`sudo chmod 740 /var/run/frr && systemctl restart frr` 

SHOW THE ROUTES VIA THE UI  `route`  
SHOW THE ROUTES VIA `vtysh`  
`show ip route`  

Select the EC2 instance named `ONPREM-SERVER2` and connect via SSM Session Manager
run `ping IP_ADDRESS_OF_EC2-B`  

Select the EC2 instance named `EC2-B`  and connect via SSM Session Manager
run `ping IP_ADDRESS_OF_ONPREM-SERVER2`  

# Finished!

And it's as easy as that. In a few steps its easy to create a dynamic VPN connection between your on-prem and AWS environments! Enjoy your connectivity!

Check out the offical documentation for an alternative way of learning https://docs.aws.amazon.com/vpn/latest/s2svpn/SetUpVPNConnections.html

