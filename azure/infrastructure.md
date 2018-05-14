   * [Setup](#setup)
   * [Networking](#networking)
      * [VPN](#vpn)
         * [OpenSSL](#openssl)
   * [Compute](#compute)
   * [Clean up](#clean-up)

# Setup
[Installation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest)  
Via a script https://azure.github.io/projects/clis/

Exercise  
https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-point-to-site-resource-manager-portal

```bash
az login
```

Set the subscription to use e.g. *Visual Studio Enterprise – MPN* or *Pay-As-You-Go*.  
If you need to move resources to a different subscription do [this](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-move-resources).
```bash
az account set -s "Visual Studio Enterprise – MPN"
```

Configure the default region/location. Make sure your location is supported by your subscription or you might get an error message when creating certain resources like *"Your subscription doesn't support virtual machine creation in North Central US. Choose a different location. Supported locations are: East US, East US 2, South Central US, West Europe, Southeast Asia, WestUS 2, West Central US."*
```bash
az account list-locations
az configure --defaults location=eastus
```

Create a resource group
```bash
az group create --name myResourceGroup
az configure --defaults group=myResourceGroup
```

# Networking

Create a virtual network and front-end subnet
```bash
az network vnet create --resource-group myResourceGroup --name myVnet --address-prefix 10.0.0.0/16 \
--subnet-name mySubnetFrontEnd --subnet-prefix 10.0.1.0/24
```

Create Network Security Groups
```bash
# myNSG
```

## VPN
Create a gateway subnet.  
The name must be "GatewaySubnet" for Azure to know this will be used for a gateway.  
When working with gateway subnets, avoid associating a network security group (NSG) to the gateway subnet. Associating a network security group to this subnet may cause your VPN gateway to stop functioning as expected.
```bash
az network vnet subnet create --address-prefix 10.0.200.0/27 --name GatewaySubnet \
--resource-group myResourceGroup --vnet-name myVnet
```

You'll need to provision a public IP address for the VPN Gateway.  
VPN Gateway currently only supports Dynamic Public IP address allocation but the only time the Public IP address changes is when the gateway is deleted and re-created.
```bash
az network public-ip create --name myVnetGWIP --resource-group myResourceGroup --allocation-method Dynamic
```

Create a virtual network gateway.  
It's typically for Site-To-Site VPNs but can be used for Point-To-Site as well.
```bash
az network vnet-gateway create --name myVnetGW --public-ip-address myVnetGWIP --resource-group myResourceGroup \
--vnet myVnet --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --no-wait
```

Certificates are used by Azure to authenticate clients connecting to a VNet over a Point-to-Site VPN connection. Once you obtain a root certificate, you upload the public key information to Azure. The root certificate is then considered 'trusted' by Azure for connection over P2S to the virtual network. You also generate client certificates from the trusted root certificate, and then install them on each client computer. The client certificate is used to authenticate the client when it initiates a connection to the VNet.

Because we're doing P2S and we only need to trust ourselves, we'll do a self-signed certificate. You can generate it with Powershell, MakeCert, or OpenSSL.

### OpenSSL
- Generate a strong private key, stored in PEM format.
```bash
openssl genrsa -aes256 -out MyAzureVPN.key 2048
 # If you needed to extract just the public key
 # openssl rsa -in MyAzureVPN.key -pubout -out MyAzureVPN-public.key
```
- Create a Certificate Signing Request (CSR) and send it to a CA. It contains the public key and is signed with the private key. In our case -x509 outputs a self-signed certificate instead of a certificate request.
```bash
 openssl req -new -x509 -days 365 -key MyAzureVPN.key -out MyAzureVPN.cer -subj /CN="MyAzureVPN"
```
- Install the CA-provided (or self-signed) certificate in your server/VPN gateway. Go to you VPN Gateway Point-to-site configuration and upload the certificate. You'll need to choose an address pool that doesn't overlap with your VNet like 172.16.201.0/24. Azure needs you to remove the "begin certificate" and "end certificate" lines and all line breaks.
- Generate and install client certificates.
```bash
# Generate private key
openssl genrsa -aes256 -out client1.key 2048
# Create certificate request
openssl req -new -out client1.req -key client1.key -subj /CN="MyAzureVPN"
# Generate a certificate from the certificate request and sign it as the CA that you are.
openssl x509 -req -in client1.req -out client1.cer -CAkey MyAzureVPN.key -CA MyAzureVPN.cer -days 1800 \
-CAcreateserial -CAserial serial
# Pack key and certificate in a .pfx (pkcs12 format)
openssl pkcs12 -export -out client1.pfx -inkey client1.key -in client1.cer -certfile MyAzureVPN.cer
```
For Windows: Double click on the .pfx to install on your Windows certificate store (certmgr).  
For [Mac OS](https://www.sslsupportdesk.com/how-to-import-a-certificate-into-mac-os/): Open Keychain Access, File, Import Items.  
For [Linux](https://superuser.com/questions/437330/how-do-you-add-a-certificate-authority-ca-to-ubuntu): Use your system's certificate [store](https://unix.stackexchange.com/questions/90450/adding-a-self-signed-certificate-to-the-trusted-list).

Download the VPN client from your virtual network gateway point-to-site configuration page. Once installed you can get to your VPN connection from the network icon on your taskbar.

# Compute
Create availability set.
```bash
az vm availability-set create \
    --name myAvailabilitySet \
    --resource-group myResourceGroup
```

Create VMs. Put them in the availability set. Specify name of Network Security Group to create or existing one to use.
```bash
az vm create \
    --resource-group myResourceGroup \
    --name myVM1 \
    --image win2016datacenter \
    --vnet-name myVnet \
    --subnet mySubnetFrontEnd \
    --availability-set myAvailabilitySet \
    --nsg myNSG
    --admin-username azureuser \
    --admin-password myPassword12
```

Create a load balancer.
```bash

```

# Clean up
Delete everything
```bash
az group delete --name myResourceGroup
```
