# Enterprise network in Azure
This is demo flow that demonstrates enterprise networking design capabilities. Key principles are:

* 3 levels network filtering hierarchy (eg. BU -> environment -> app) with top level being VNETs filtered via network virtual appliances, second level subnetes inside VNET filteret with NSG and microsegmentation on VM level filtered with NSG/ASG
* one shared gateway per region to on-premises network (eg. service VNET managed by networking/security team), shared infrastructure environment (DNS, AD, ...)
* BU VNETs forced to use service VNET as gateway
* Cross BU and cross site communication forced via virtual appliance
* All Internet access to go via on-premises (security requirement)
* Inter-region communication to leverage Microsoft network (not forcing traffic via on-premises)
* Gateway to on-premises in all used regions, shortest path to cloud

# Table of Contents
- [Enterprise network in Azure](#enterprise-network-in-azure)
- [Table of Contents](#table-of-contents)
- [Connected networks in Azure](#connected-networks-in-azure)
    - [Store target regions in environment variable](#store-target-regions-in-environment-variable)
    - [Create Resource Group](#create-resource-group)
    - [Create networks](#create-networks)
        - [Deploy Cisco CSR to simulate on-premises environment](#deploy-cisco-csr-to-simulate-on-premises-environment)
        - [Region 1](#region-1)
            - [Hub](#hub)
            - [Spoke1](#spoke1)
            - [Spoke2](#spoke2)
        - [Region 2](#region-2)
            - [Hub](#hub)
            - [Spoke1](#spoke1)
            - [Spoke2](#spoke2)
    - [Configure VPN gateways and BGP routing](#configure-vpn-gateways-and-bgp-routing)
        - [Create public IP for VPN endpoints](#create-public-ip-for-vpn-endpoints)
        - [Deploy Azure VPN](#deploy-azure-vpn)
        - [Configure Azure VPN peering and connection to on-premises CSR](#configure-azure-vpn-peering-and-connection-to-on-premises-csr)
        - [Configure on-premises router](#configure-on-premises-router)
            - [Generate IPSEC configuration and send it to CSR](#generate-ipsec-configuration-and-send-it-to-csr)
            - [Check IPSec connection](#check-ipsec-connection)
            - [Generate routing configuration and send it to CSR](#generate-routing-configuration-and-send-it-to-csr)
        - [Check routing](#check-routing)
    - [Configure intra-region peering](#configure-intra-region-peering)
        - [Get network IDs](#get-network-ids)
        - [Configure spokes to hub](#configure-spokes-to-hub)
        - [Configure hubs to spoke](#configure-hubs-to-spoke)
        - [Check routing on on-premises router](#check-routing-on-on-premises-router)
        - [Check routing in Azure](#check-routing-in-azure)
            - [Create NIC in all VNETs and subnets](#create-nic-in-all-vnets-and-subnets)
            - [Run small Linux VMs](#run-small-linux-vms)
            - [List effective routes](#list-effective-routes)
- [Deploying virtual network device into service VNET to filter and route traffic between BUs](#deploying-virtual-network-device-into-service-vnet-to-filter-and-route-traffic-between-bus)
    - [Deploy CSR into hub VNETs](#deploy-csr-into-hub-vnets)
    - [Setup routing in spoke VNETs to point to our CSR deployed in hub](#setup-routing-in-spoke-vnets-to-point-to-our-csr-deployed-in-hub)
        - [Create route tables](#create-route-tables)
        - [Add routes to tables](#add-routes-to-tables)
        - [Associate tables with spoke subnets](#associate-tables-with-spoke-subnets)
        - [Check effective routes on one of NICs in spoke VNET](#check-effective-routes-on-one-of-nics-in-spoke-vnet)
        - [Connect to CSR in hub and enable gigabiethernet 2 interface](#connect-to-csr-in-hub-and-enable-gigabiethernet-2-interface)
        - [Test connectivity and control from on BU to another](#test-connectivity-and-control-from-on-bu-to-another)
        - [Making sure traffic between regions hit BOTH virtual network security devices (CSR in our case)](#making-sure-traffic-between-regions-hit-both-virtual-network-security-devices-csr-in-our-case)
- [Enable Internet access for BUs via network appliance in hub VNET](#enable-internet-access-for-bus-via-network-appliance-in-hub-vnet)
    - [Configure route for csrext subnet to access Internet via Azure](#configure-route-for-csrext-subnet-to-access-internet-via-azure)
    - [Configure CSR to enable Internet access](#configure-csr-to-enable-internet-access)
    - [Test access to Internet](#test-access-to-internet)
- [Configure intra-region peering](#configure-intra-region-peering)
- [Add NSG on subnet level and NSG/ASG microsegmentation on NIC level](#add-nsg-on-subnet-level-and-nsgasg-microsegmentation-on-nic-level)


# Connected networks in Azure
Let's start with schema.

![alt text](https://raw.githubusercontent.com/tkubica12/azure-enterprise-networking/master/images/Drawing1.jpg "Schema 1")

## Store target regions in environment variable
For our demo we gonna need two regions. We will use global VNET peering feature which is currently in Preview and limited to certain regions. For now we will select following two regions.

```
export reg1='westcentralus'
export reg2='westus2'
```

## Create Resource Group
Create resource group for our networks. For simplicity of this demo we will have all networking in single group so its location will be overide for resources we will push. In more real life scenario those environments will be managed in different resource groups to enable proper RBAC or even in separated subscriptions (like central team subscription for hub and separate BU subscriptions for spokes).

```
az group create -n peering -l westeurope
```

## Create networks
First we will create networks (VNETs) and subnets. We want one hub VNET (service VNET) that will host network appliance, shared services like AD, DNS, LDAP, WSUS, Linux repo etc. and gateway to on-premises network. This service VNET is managed by networking and security team, application owners and BUs have no access to it. For BUs separate VNETs will be created with no individual access to on-premises, but rather with forced flow via service VNET. BU (spoke) VNETs will have multiple subnets (eg. environments).

### Deploy Cisco CSR to simulate on-premises environment

In order to have "on-premises" network in our demo we are going to use Cisco CSR 1000v that will server as VPN connection to both Azure cloud regions, BGP peer etc. 

If you are deploying CSR from market place for first time you actualy need to go throw GUI wizard first so you accept vendor terms and conditions. Use whatever parameters you like and then simply delete those resources. Afterwards in your subscription you do not have to accept those every time so we can use ARM template to deploy this programatically. 

Beware that template does including public SSH key to authenticate to your router! In order for you to be able to connect please use public key of your own private key (change value in csr-parameters.json). Also make sure other parameters are modified as well - especially storage account and DNS name as those need to be globaly unique.

```
az group create -n peering-onprem -l westeurope
az group deployment create --name csr --resource-group peering-onprem --template-file csr-template.json --parameters @csr-parameters-onprem.json
```

### Region 1
#### Hub
```
az network vnet create -g peering -n netreg1hub --address-prefix 10.100.0.0/16 --subnet-name netreg1hubsub1 --subnet-prefix 10.100.0.0/24 -l $reg1
az network vnet subnet create -g peering --vnet-name netreg1hub -n GatewaySubnet --address-prefix 10.100.254.0/24
```

#### Spoke1
```
az network vnet create -g peering -n netreg1spoke1 --address-prefix 10.101.0.0/16 --subnet-name netreg1spoke1sub1 --subnet-prefix 10.101.1.0/24 -l $reg1
az network vnet subnet create -g peering --vnet-name netreg1spoke1 -n netreg1spoke1sub2 --address-prefix 10.101.2.0/24
```

#### Spoke2
```
az network vnet create -g peering -n netreg1spoke2 --address-prefix 10.102.0.0/16 --subnet-name netreg1spoke2sub1 --subnet-prefix 10.102.1.0/24 -l $reg1
az network vnet subnet create -g peering --vnet-name netreg1spoke2 -n netreg1spoke2sub2 --address-prefix 10.102.2.0/24
```

### Region 2
#### Hub
```
az network vnet create -g peering -n netreg2hub --address-prefix 10.200.0.0/16 --subnet-name netreg2hubsub1 --subnet-prefix 10.200.0.0/24 -l $reg2
az network vnet subnet create -g peering --vnet-name netreg2hub -n GatewaySubnet --address-prefix 10.200.254.0/24
```

#### Spoke1
```
az network vnet create -g peering -n netreg2spoke1 --address-prefix 10.201.0.0/16 --subnet-name netreg2spoke1sub1 --subnet-prefix 10.201.1.0/24 -l $reg2
az network vnet subnet create -g peering --vnet-name netreg2spoke1 -n netreg2spoke1sub2 --address-prefix 10.201.2.0/24
```

#### Spoke2
```
az network vnet create -g peering -n netreg2spoke2 --address-prefix 10.202.0.0/16 --subnet-name netreg2spoke2sub1 --subnet-prefix 10.202.1.0/24 -l $reg2
az network vnet subnet create -g peering --vnet-name netreg2spoke2 -n netreg2spoke2sub2 --address-prefix 10.202.2.0/24
```

## Configure VPN gateways and BGP routing
Now we need to connect our two cloud regions with on-premises network (represented by our CSR). We will estabilish two IPSec tunnels (one for each region) and run BGP inside this tunnel. Cloud will by AS 65001, our on-premises is 65002. On-premises environment is our CSR while in Azure we will use standard Azure VPN (managed VPN service).

### Create public IP for VPN endpoints
IPSec will be estabilished over Internet so we first need to create public IP address in each region that will be used as Azure VPN endpoint.

```
az network public-ip create -n reg1ip -g peering --allocation-method dynamic -l $reg1
az network public-ip create -n reg2ip -g peering --allocation-method dynamic -l $reg2
```

### Deploy Azure VPN
Create Azure VPN in each region with public IP and 65001 BGP AS number. Please note that this operation can take about 30 minutes.

```
az network vnet-gateway create -n reg1vpn --public-ip-address reg1ip -g peering --vnet netreg1hub -l $reg1 --sku VpnGw1 --asn 65001
az network vnet-gateway create -n reg2vpn --public-ip-address reg2ip -g peering --vnet netreg2hub -l $reg2 --sku VpnGw1 --asn 65001
```

### Configure Azure VPN peering and connection to on-premises CSR
In this step we will configure Azure VPN IPSec connection and BGP peering.

Get CSR IP and configure in Azure VPN. Please note in Azure terminology "local gateway" is your on premises router. From networking perspective I would rather call that "remote peer" as from Azure VPN configuration point of view this is IP (and other settings) of peer your are connecting to.

```
export ciscoIP=$(az network public-ip show -n mycsrip -g peering-onprem --query ipAddress -o tsv)
export ciscoLoopback='10.254.254.1'
az network local-gateway create -n csrpeeringreg1 -g peering --gateway-ip-address $ciscoIP --asn 65002 --bgp-peering-address $ciscoLoopback -l $reg1
az network local-gateway create -n csrpeeringreg2 -g peering --gateway-ip-address $ciscoIP --asn 65002 --bgp-peering-address $ciscoLoopback -l $reg2
```

Create new VPN connection in Azure VPN with BGP enabled.
```
az network vpn-connection create -n vpnreg1toCSRconnection -g peering --vnet-gateway1 reg1vpn --enable-bgp -l $reg1 --shared-key Azure12345678 --local-gateway csrpeeringreg1
az network vpn-connection create -n vpnreg2toCSRconnection -g peering --vnet-gateway1 reg2vpn --enable-bgp -l $reg2 --shared-key Azure12345678 --local-gateway csrpeeringreg2
```

### Configure on-premises router
We will configure CSR to connect to Azure. You can access router at following address, but in following sections we will use templates to generate configuration script and pipe it directly to router (so no need to manualy operate in router CLI).

```
ssh tomas@$ciscoIP
```

#### Generate IPSEC configuration and send it to CSR
In file csr_ipsec.template there is example CSR configuration. We will take IP addresses and use sed to build final configuration from this template and pipe it to CSR.

```
export reg1vpnip=$(az network public-ip show -n reg1ip -g peering --query ipAddress -o tsv)
export reg2vpnip=$(az network public-ip show -n reg2ip -g peering --query ipAddress -o tsv)
sed -e 's/{localip}/'$ciscoIP'/' -e 's/{remoteip1}/'$reg1vpnip'/' -e 's/{remoteip2}/'$reg2vpnip'/' csr_ipsec.template | ssh tomas@$ciscoIP
```

#### Check IPSec connection
After some time we should see vpn-connection status in Azure to change from Connecting to Connected.

```
az network vpn-connection show -n vpnreg1toCSRconnection -g peering --query connectionStatus
az network vpn-connection show -n vpnreg2toCSRconnection -g peering --query connectionStatus
```

#### Generate routing configuration and send it to CSR
Now IPSec is up we need to configure routing. Specificaly ensure we can reach BGP peering loopback in Azure (we will use static route for that) and configure BGP to start exchanging routes. Please note that to fullfil security requirements we need to advertise default route via BGP so network in Azure do not access internet directly, only throw on-premises.

```
export reg1peeringip=$(az network vnet-gateway show -n reg1vpn -g peering --query bgpSettings.bgpPeeringAddress -o tsv)
export reg2peeringip=$(az network vnet-gateway show -n reg2vpn -g peering --query bgpSettings.bgpPeeringAddress -o tsv)
sed -e 's/{reg1peeringip}/'$reg1peeringip'/' -e 's/{reg2peeringip}/'$reg2peeringip'/'  csr_routing.template | ssh tomas@$ciscoIP
```

### Check routing
Our on-premises router should now see to prefixes learned via BGP - on pointing to vpn gateway in reg1 and one to vpn gateway in reg2.

```
ssh tomas@$ciscoIP 'show ip route bgp'
```

You should see the following BGP routing table:

```
B        10.100.0.0/16 [20/0] via 10.100.254.254, 00:10:35
B        10.200.0.0/16 [20/0] via 10.200.254.254, 00:09:40
```

## Configure intra-region peering
We have connected on-premises environment with both Azure regions, but only to service VNET (hub network). In order for BU VNETs to be able to access shared resources and on-premises we need to configure VNET peering between hub and spoke VNETs.

### Get network IDs
```
export netreg1hub=$(az network vnet show -g peering -n netreg1hub --query id -o tsv)
export netreg1spoke1=$(az network vnet show -g peering -n netreg1spoke1 --query id -o tsv)
export netreg1spoke2=$(az network vnet show -g peering -n netreg1spoke2 --query id -o tsv)
export netreg2hub=$(az network vnet show -g peering -n netreg2hub --query id -o tsv)
export netreg2spoke1=$(az network vnet show -g peering -n netreg2spoke1 --query id -o tsv)
export netreg2spoke2=$(az network vnet show -g peering -n netreg2spoke2 --query id -o tsv)
```

### Configure spokes to hub
```
az network vnet peering create -n reg1-spoke1-hub -g peering --remote-vnet-id $netreg1hub --vnet-name netreg1spoke1 --allow-forwarded-traffic --allow-vnet-access --use-remote-gateways
az network vnet peering create -n reg1-spoke2-hub -g peering --remote-vnet-id $netreg1hub --vnet-name netreg1spoke2 --allow-forwarded-traffic --allow-vnet-access --use-remote-gateways
az network vnet peering create -n reg2-spoke1-hub -g peering --remote-vnet-id $netreg2hub --vnet-name netreg2spoke1 --allow-forwarded-traffic --allow-vnet-access --use-remote-gateways
az network vnet peering create -n reg2-spoke2-hub -g peering --remote-vnet-id $netreg2hub --vnet-name netreg2spoke2 --allow-forwarded-traffic --allow-vnet-access --use-remote-gateways
```

### Configure hubs to spoke
```
az network vnet peering create -n reg1-hub-spoke1 -g peering --remote-vnet-id $netreg1spoke1 --vnet-name netreg1hub --allow-forwarded-traffic --allow-vnet-access --allow-gateway-transit
az network vnet peering create -n reg1-hub-spoke2 -g peering --remote-vnet-id $netreg1spoke2 --vnet-name netreg1hub --allow-forwarded-traffic --allow-vnet-access --allow-gateway-transit
az network vnet peering create -n reg2-hub-spoke1 -g peering --remote-vnet-id $netreg2spoke1 --vnet-name netreg2hub --allow-forwarded-traffic --allow-vnet-access --allow-gateway-transit
az network vnet peering create -n reg2-hub-spoke2 -g peering --remote-vnet-id $netreg2spoke2 --vnet-name netreg2hub --allow-forwarded-traffic --allow-vnet-access --allow-gateway-transit
```

### Check routing on on-premises router
Our on-premises router should now see prefixes to all VNETs.

```
ssh tomas@$ciscoIP 'show ip route bgp'
```

You should see the following BGP routing table:

```
B        10.100.0.0/16 [20/0] via 10.100.254.254, 00:10:35
B        10.101.0.0/16 [20/0] via 10.100.254.254, 00:01:43
B        10.102.0.0/16 [20/0] via 10.100.254.254, 00:01:28
B        10.200.0.0/16 [20/0] via 10.200.254.254, 00:09:40
B        10.201.0.0/16 [20/0] via 10.200.254.254, 00:01:09
B        10.202.0.0/16 [20/0] via 10.200.254.254, 00:00:57
```

### Check routing in Azure
Azure software defined networking is distributed in nature so we are not able to look at routing on subnet level for example. Because routing is distributed to hosts we need to create VM  so we can look at effective routing tables.

#### Create NIC in all VNETs and subnets
First let's create NIC in all VNETs and subnets

```
az network nic create -g peering --vnet-name netreg1hub --subnet netreg1hubsub1 -n nicreg1hubsub1 -l $reg1
az network nic create -g peering --vnet-name netreg1spoke1 --subnet netreg1spoke1sub1 -n nicreg1spoke1sub1 -l $reg1
az network nic create -g peering --vnet-name netreg1spoke1 --subnet netreg1spoke1sub2 -n nicreg1spoke1sub2 -l $reg1
az network nic create -g peering --vnet-name netreg1spoke2 --subnet netreg1spoke2sub1 -n nicreg1spoke2sub1 -l $reg1
az network nic create -g peering --vnet-name netreg1spoke1 --subnet netreg1spoke1sub2 -n nicreg1spoke2sub2 -l $reg1
az network nic create -g peering --vnet-name netreg2hub --subnet netreg2hubsub1 -n nicreg2hubsub1 -l $reg2
az network nic create -g peering --vnet-name netreg2spoke1 --subnet netreg2spoke1sub1 -n nicreg2spoke1sub1 -l $reg2
az network nic create -g peering --vnet-name netreg2spoke1 --subnet netreg2spoke1sub2 -n nicreg2spoke1sub2 -l $reg2
az network nic create -g peering --vnet-name netreg2spoke2 --subnet netreg2spoke2sub1 -n nicreg2spoke2sub1 -l $reg2
az network nic create -g peering --vnet-name netreg2spoke1 --subnet netreg2spoke1sub2 -n nicreg2spoke2sub2 -l $reg2
```

#### Run small Linux VMs
Create VMs. For our testing purposes we do not need to analyze all possible subnets. Let's create VMs in hub (service VNET) in both regions, in region 1 one VM in each subnet os one BU (one spoke VNET) and one in one subnet of other BU, in region 2 one VM in one BU subnet.

```
az vm create -n nicreg1hubsub1 -g peering --image UbuntuLTS --size Basic_A0 --admin-username tomas --admin-password Azure12345678 --authentication-type password --nics nicreg1hubsub1 -l $reg1 --no-wait
az vm create -n nicreg1spoke1sub1 -g peering --image UbuntuLTS --size Basic_A0 --admin-username tomas --admin-password Azure12345678 --authentication-type password --nics nicreg1spoke1sub1 -l $reg1 --no-wait
az vm create -n nicreg1spoke1sub2 -g peering --image UbuntuLTS --size Basic_A0 --admin-username tomas --admin-password Azure12345678 --authentication-type password --nics nicreg1spoke1sub2 -l $reg1 --no-wait
az vm create -n nicreg1spoke2sub1 -g peering --image UbuntuLTS --size Basic_A0 --admin-username tomas --admin-password Azure12345678 --authentication-type password --nics nicreg1spoke2sub1 -l $reg1 --no-wait
az vm create -n nicreg2hubsub1 -g peering --image UbuntuLTS --size Basic_A0 --admin-username tomas --admin-password Azure12345678 --authentication-type password --nics nicreg2hubsub1 -l $reg2 --no-wait
az vm create -n nicreg2spoke1sub1 -g peering --image UbuntuLTS --size Basic_A0 --admin-username tomas --admin-password Azure12345678 --authentication-type password --nics nicreg2spoke1sub1 -l $reg2 --no-wait
```
#### List effective routes
First check effective routes in our hub VNETs.

```
az network nic show-effective-route-table -n nicreg1hubsub1 -g peering
az network nic show-effective-route-table -n nicreg2hubsub1 -g peering
```

What we expect? In region 1 there should be routes like this:
* 10.100.0.0/16 -> VnetLocal
* 10.101.0.0/16 -> VnetPeering
* 10.102.0.0/16 -> VnetPeering
* 10.254.254.1/32 -> VirtualNetworkGateway, NextHop <yourAzureGatewayPublicIP>
* 0.0.0.0/0 -> VirtualNetworkGateway, NextHop <yourAzureGatewayPublicIP> (= our on-premises router has advertised default route to Azure)
* Blackhole to other private addresses (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10)

In region 2 we should see:
* 10.200.0.0/16 -> VnetLocal
* 10.201.0.0/16 -> VnetPeering
* 10.202.0.0/16 -> VnetPeering
* 10.254.254.1/32 -> VirtualNetworkGateway, NextHop <yourAzureGatewayPublicIP>
* 0.0.0.0/0 -> VirtualNetworkGateway, NextHop <yourAzureGatewayPublicIP> (= our on-premises router has advertised default route to Azure)
* Blackhole to other private addresses (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 100.64.0.0/10)

Let's investigate routing information in NIC that is in BU VNET.

```
az network nic show-effective-route-table -n nicreg1spoke1sub1 -g peering
az network nic show-effective-route-table -n nicreg1spoke1sub2 -g peering
...
```

As we can see routing inside VNET is distributed function and is not done on subnet level. In other words subnets are objects that help us define security rules and categories IP addresses, but not important for routing. All traffic within VNET is basically always L3 (there is no L2 in Azure - no broadcasts etc.). Therefore effective routes on nicreg1spoke1sub1 are the same as nicreg1spoke1sub2.

We can also make sure we are able to ping from on-premises router to out VMs.

```
ssh tomas@$ciscoIP 'ping 10.100.0.4 source 10.254.254.1'
ssh tomas@$ciscoIP 'ping 10.200.0.4 source 10.254.254.1'
ssh tomas@$ciscoIP 'ping 10.101.1.4 source 10.254.254.1'
ssh tomas@$ciscoIP 'ping 10.101.2.4 source 10.254.254.1'
ssh tomas@$ciscoIP 'ping 10.102.1.4 source 10.254.254.1'
```


# Deploying virtual network device into service VNET to filter and route traffic between BUs
First let's visualize what we want to achive.

![alt text](https://raw.githubusercontent.com/tkubica12/azure-enterprise-networking/master/images/Drawing2.jpg "Schema 2")

In this section we will deploy virtual network applicance in service VNET (hub) so we have better control from network security perspective. In our example this appliance will only be Cisco CSR 1000v router (for demonstration purposes), but you can easily have complex NGFW device deployed instead to implement deeper functionality such as IPS, antitvirus, DLP, web gateway etc. 

At this point spoke VNETs are able to communicate with hub network (service VNET), for example to access shared infrastructure services such as DNS, AD or LDAP, and also leverage its peering connections to talk to other BUs (spoke VNETs). If we trace communicaiton flow from spoke1 to spoke2 we should se traffic goes throw hub VNET.

```
ssh tomas@$ciscoIP 'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ tracepath 10.102.1.4
tracepath 10.102.1.4
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.100.254.4                                          1.060ms
 1:  10.100.254.4                                          0.851ms
 2:  10.102.1.4                                            2.529ms reached
     Resume: pmtu 1500 hops 2 back 2
```

## Deploy CSR into hub VNETs
We will use same template as before to deploy CSR, but with different parameters file. You can use files csr-parameters-reg1.json and csr-parameters-reg2.json, just make sure  your storage account and DNS names are globaly unique. This template deploys CSR with password authentication - I have used Azure12345678. For purpose of this demo I consider region definition variable, so let's override location parameter via CLI. We will also create two additional subnets. One called link in which we will have one interface of CSR and also dmz subnet as second interface.

```
az group create -n peering-reg1-csr -l $reg1
az network vnet subnet create -g peering --vnet-name netreg1hub -n csrext --address-prefix 10.100.10.0/24
az network vnet subnet create -g peering --vnet-name netreg1hub -n csrint --address-prefix 10.100.11.0/24
az group deployment create --name csr --resource-group peering-reg1-csr --template-file csr-template.json --parameters @csr-parameters-reg1.json --parameters location=$reg1

az group create -n peering-reg2-csr -l $reg2
az network vnet subnet create -g peering --vnet-name netreg2hub -n csrext --address-prefix 10.200.10.0/24
az network vnet subnet create -g peering --vnet-name netreg2hub -n csrint --address-prefix 10.200.11.0/24
az group deployment create --name csr --resource-group peering-reg2-csr --template-file csr-template.json --parameters @csr-parameters-reg2.json --parameters location=$reg2
```

Get our CSR IPs.

```
export csrreg1IP=$(az network public-ip show -n mycsripreg1 -g peering-reg1-csr --query ipAddress -o tsv)
export csrreg2IP=$(az network public-ip show -n mycsripreg2 -g peering-reg2-csr --query ipAddress -o tsv)
export csrreg1Private=$(az network nic show -n csr-Nic1 -g peering-reg1-csr --query ipConfigurations[0].privateIpAddress -o tsv)
export csrreg2Private=$(az network nic show -n csr-Nic1 -g peering-reg2-csr --query ipConfigurations[0].privateIpAddress -o tsv)
```

## Setup routing in spoke VNETs to point to our CSR deployed in hub
For our solution we will want to send all traffic that goes out of spoke VNET to hit CSR deployed in hub VNET.

### Create route tables
```
az network route-table create -n routeToHubApplianceReg1 -g peering -l $reg1
az network route-table create -n routeToHubApplianceReg2 -g peering -l $reg2
```

### Add routes to tables
```
az network route-table route create -n defaultToAppliance --address-prefix '0.0.0.0/0' --next-hop-type VirtualAppliance -g peering --route-table-name routeToHubApplianceReg1 --next-hop-ip-address $csrreg1Private

az network route-table route create -n defaultToAppliance --address-prefix '0.0.0.0/0' --next-hop-type VirtualAppliance -g peering --route-table-name routeToHubApplianceReg2 --next-hop-ip-address $csrreg2Private
```

### Associate tables with spoke subnets
```
az network vnet subnet update -n netreg1spoke1sub1 --vnet-name netreg1spoke1 -g peering --route-table routeToHubApplianceReg1
az network vnet subnet update -n netreg1spoke1sub2 --vnet-name netreg1spoke1 -g peering --route-table routeToHubApplianceReg1
az network vnet subnet update -n netreg1spoke2sub1 --vnet-name netreg1spoke2 -g peering --route-table routeToHubApplianceReg1
az network vnet subnet update -n netreg1spoke2sub2 --vnet-name netreg1spoke2 -g peering --route-table routeToHubApplianceReg1

az network vnet subnet update -n netreg2spoke1sub1 --vnet-name netreg2spoke1 -g peering --route-table routeToHubApplianceReg2
az network vnet subnet update -n netreg2spoke1sub2 --vnet-name netreg2spoke1 -g peering --route-table routeToHubApplianceReg2
az network vnet subnet update -n netreg2spoke2sub1 --vnet-name netreg2spoke2 -g peering --route-table routeToHubApplianceReg2
az network vnet subnet update -n netreg2spoke2sub2 --vnet-name netreg2spoke2 -g peering --route-table routeToHubApplianceReg2
```

### Check effective routes on one of NICs in spoke VNET
```
az network nic show-effective-route-table -n nicreg1spoke1sub1 -g peering
```

We should see previous default route (0.0.0.0/0) pointing to gateway in hub VNET should be in state invalid as it is replaced by our custom route. That is pointing to our virtual appliance.

### Connect to CSR in hub and enable gigabiethernet 2 interface
CSR by comes with GigabitEthernet 2 interface shut down. Because we use this interface in csrint subnet as gateway for our BU VMs, we need to connect to CSR and enable it. Currently our CSR does not have direct access to Internet, so let's connect via our on-premises router using CSR IP on first NIC.

```
ssh tomas@$ciscoIP 'ssh 10.100.10.4'

config term
  interface GigabitEthernet2
    no shut
    ip address dhcp
    exit
```

And also Region 2

```
ssh tomas@$ciscoIP 'ssh 10.200.10.4'

config term
  interface GigabitEthernet2
    no shut
    ip address dhcp
    exit
```

### Test connectivity and control from on BU to another
We can now make sure our traffic realy goes via our network security appliance. When we started this chapter we did tracepath from 10.101.1.4 to 10.102.1.4 and we have seen traffic going via hub VNET SDN stack. After configuration of CSR as virtual appliance and pointing defaults from spokes to it we should se communication passing via our CSR.

```
ssh tomas@$ciscoIP 'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ tracepath 10.102.1.4
tracepath 10.102.1.4
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.100.11.4                                           2.337ms
 1:  10.100.11.4                                           1.424ms
 2:  10.102.1.4                                            2.636ms reached
     Resume: pmtu 1500 hops 2 back 2
```

In order to prove that stop CSR VM and check traffic is no longer going throw.

```
az vm stop -n csr -g peering-reg1-csr
ssh tomas@$ciscoIP 'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ ping 10.102.1.4
az vm start -n csr -g peering-reg1-csr
```

You can also see that our communication between regions first go via our CRS in hub VNET (10.100.11.4) and than continues via VPN tunnel to on-premises router (52.166.193.101) and from there to second region (in following chapters we will look into how to make this communication to go directly between clouds with Global VNET peering).

```
ssh tomas@$ciscoIP 'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ tracepath 10.201.1.4
tracepath 10.201.1.4
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.100.11.4                                           2.073ms
 1:  10.100.11.4                                           1.139ms
 2:  10.100.254.5                                          0.967ms asymm  1
 3:  10.100.254.5                                          0.972ms pmtu 1400
 3:  52.166.193.101                                      136.442ms asymm  2
 4:  no reply
 5:  10.201.1.4                                          294.122ms reached
     Resume: pmtu 1400 hops 5 back 5
```

### Making sure traffic between regions hit BOTH virtual network security devices (CSR in our case)
As we saw in our previous tracepath traffic from BU goes via local security device (CSR), than continues to on-premises device via VPN, than to secondary region and from there it takes direct path to other BU VM. In other words returning traffic is not seen by CSR. This might be issue for some security devices as session state cannot be tracked. Let's now modify our routing so that traffic comming from gateway to spokes is always routed to our CSR in hub VNET.

```
az network route-table create -n routeSpokesViaApplianceReg1 -g peering -l $reg1
az network route-table create -n routeSpokesViaApplianceReg2 -g peering -l $reg2

az network route-table route create -n toSpoke1ViaAppliance --address-prefix '10.101.0.0/16' --next-hop-type VirtualAppliance -g peering --route-table-name routeSpokesViaApplianceReg1 --next-hop-ip-address $csrreg1Private
az network route-table route create -n toSpoke2ViaAppliance --address-prefix '10.102.0.0/16' --next-hop-type VirtualAppliance -g peering --route-table-name routeSpokesViaApplianceReg1 --next-hop-ip-address $csrreg1Private

az network route-table route create -n toSpoke1ViaAppliance --address-prefix '10.201.0.0/16' --next-hop-type VirtualAppliance -g peering --route-table-name routeSpokesViaApplianceReg2 --next-hop-ip-address $csrreg2Private
az network route-table route create -n toSpoke2ViaAppliance --address-prefix '10.202.0.0/16' --next-hop-type VirtualAppliance -g peering --route-table-name routeSpokesViaApplianceReg2 --next-hop-ip-address $csrreg2Private

az network vnet subnet update -n GatewaySubnet --vnet-name netreg1hub -g peering --route-table routeSpokesViaApplianceReg1
az network vnet subnet update -n GatewaySubnet --vnet-name netreg2hub -g peering --route-table routeSpokesViaApplianceReg2
```

Now we can test that. Compare current tracepath with situation before. As we can see traffic groes also throw security device in destination (10.200.11.4).

```
ssh tomas@$ciscoIP 'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ tracepath 10.201.1.4
tracepath 10.201.1.4
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.100.11.4                                           2.343ms
 1:  10.100.11.4                                           2.829ms
 2:  10.100.254.5                                          1.751ms asymm  1
 3:  10.100.254.5                                          1.370ms pmtu 1400
 3:  52.166.193.101                                      134.407ms asymm  2
 4:  no reply
 5:  10.200.11.4                                         293.474ms asymm  4
 6:  10.201.1.4                                          292.251ms reached
     Resume: pmtu 1400 hops 6 back 5
```

# Enable Internet access for BUs via network appliance in hub VNET
Again we are going to start with schema.

![alt text](https://raw.githubusercontent.com/tkubica12/azure-enterprise-networking/master/images/Drawing3.jpg "Schema 3")

In our current setup all Internet traffic goes back to on-premises environment. That might be desired situation for security and compliance reasons. Our demo environment on-prem simulated CSR is not configured for providing Internet access, but let's show that traffic is indeed routed there. Use tracepath from BU VM to some public IP address in Internet.

```
ssh tomas@$ciscoIP 'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ tracepath 208.67.222.222
tracepath 208.67.222.222
 1?: [LOCALHOST]                                         pmtu 1500
 1:  10.100.11.4                                           2.148ms
 1:  10.100.11.4                                           1.191ms
 2:  10.100.254.4                                          1.373ms asymm  1
 3:  10.100.254.4                                          1.748ms pmtu 1400
 3:  52.166.193.101                                      137.663ms asymm  2
 4:  no reply
```

52.166.193.101 is IP address of our on-premises router. Traffic goes to our hub CSR and from there to hub VNET and via VPN to our on-premises CSR.

In this section we still won't provide direct Internet access to spoke network, but it is ok for us from security perspective to provide this access via our enterprise grade security device deployed in hub (in our case just CSR), so we have this under full control.

## Configure route for csrext subnet to access Internet via Azure
In order to enable CSR to access Internet let's first see current effective routes on its first NIC.

```
az network nic show-effective-route-table -n csr-Nic0 -g peering-reg1-csr
```

We can see route to 0.0.0.0/0 being type VirtualNetworkGateway going to our VPN gateway. Let's add custom route to send traffic from CSR in hub VNET to Internet.

```
az network route-table create -n routeToInternetReg1 -g peering -l $reg1
az network route-table create -n routeToInternetReg2 -g peering -l $reg2

az network route-table route create -n defaultToInternet --address-prefix '0.0.0.0/0' --next-hop-type Internet -g peering --route-table-name routeToInternetReg1
az network route-table route create -n defaultToInternet --address-prefix '0.0.0.0/0' --next-hop-type Internet -g peering --route-table-name routeToInternetReg2

az network vnet subnet update -n csrext --vnet-name netreg1hub -g peering --route-table routeToInternetReg1
az network vnet subnet update -n csrext --vnet-name netreg2hub -g peering --route-table routeToInternetReg2
```

## Configure CSR to enable Internet access
Our CSR in hub VNET is receiving traffic now, but we need to configure NAT to translate Internet traffic to CSR public IP (we have already configured Azure to communicate from csrext subnet directly to Internet, not via VPN gateway, so we can manage our CSR now via its public IP - in real world scenario you might want to expose public IP on one interface without inbound management capability while using internet access on different one). We will configure NAT policy and also route to whole 10.0.0.0/8 prefix (we want this to go via csrint interface where 0.0.0.0/0 is virtual gateway - Azure VPN).

This is how we configure CSR in Region 1:

```
ssh tomas@$csrreg1IP
config term
ip access-list extended 100
  permit ip 192.168.35.1 0.0.0.0 any
  deny ip any 10.0.0.0 0.255.255.255
  deny ip any 192.168.0.0 0.0.255.255
  deny ip any 172.16.0.0 0.15.255.255
  permit ip 10.101.0.0 0.0.255.255 any
  permit ip 10.102.0.0 0.0.255.255 any
  exit

interface gigabitethernet 2
  ip nat inside
  exit

ip nat inside source list 100 interface GigabitEthernet1 overload

ip route 10.0.0.0 255.0.0.0 10.100.11.1
```

This is how we configure CSR in Region 2:

```
ssh tomas@$csrreg2IP
config term
ip access-list extended 100
  permit ip 192.168.35.1 0.0.0.0 any
  deny ip any 10.0.0.0 0.255.255.255
  deny ip any 192.168.0.0 0.0.255.255
  deny ip any 172.16.0.0 0.15.255.255
  permit ip 10.201.0.0 0.0.255.255 any
  permit ip 10.202.0.0 0.0.255.255 any
  exit

interface gigabitethernet 2
  ip nat inside
  exit

ip nat inside source list 100 interface GigabitEthernet1 overload

ip route 10.0.0.0 255.0.0.0 10.200.11.1
```


## Test access to Internet
For testing we cannot use tracepath right now as CSR in path does not allow for such tracing in our configuration (you should be able to fix that if you like). So we will just test this with ping.

Once again connect to BU VM via on-premises router and try to access Internet.

```
ssh tomas@$ciscoIP  'ssh 10.101.1.4'
tomas@nicreg1spoke1sub1:~$ ping 208.67.222.222
PING 208.67.222.222 (208.67.222.222) 56(84) bytes of data.
64 bytes from 208.67.222.222: icmp_seq=1 ttl=49 time=28.7 ms
64 bytes from 208.67.222.222: icmp_seq=2 ttl=49 time=28.8 ms
...
```

# Configure intra-region peering
TODO

# Add NSG on subnet level and NSG/ASG microsegmentation on NIC level
TODO