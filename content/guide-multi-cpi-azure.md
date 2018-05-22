!!! note
    BOSH supports Multi-CPI since version v261+.

In this guide we explore how to configure BOSH to deploy VMs from a single deployment across two different Azure regions with two seperate Azure service principals. Communication between regions will be configured via VNet-to-VNet VPN gateway.

---
## Set up the IaaS {: #setup-iaas }

Let's start by initializing main AZ (`z1`) to `West Europe` by following steps 1 and 2 from [Creating environment on Azure](init-azure.md). This will give you a working BOSH Director in a single region. You can perform a deployment to test Director is working fine. Then create a loader balancer with Standard SKU using template [here](https://github.com/Azure/azure-quickstart-templates/blob/master/bosh-setup/nestedtemplates/load-balancer-availability-zones-enabled.json), you can learn how to create resource with template [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-deploy-portal#deploy-resources-from-custom-template). Remember the name and public IP address of the load balancer, it will be used later.

To add a second AZ (`z2`) to `East US 2` you need to perform the following actions with another Azure service principals.

* Create a new resource group in `East US 2`.
* Create a Virtual Network  named "boshvnet-crp"(10.1.16.0/20) in created resourse group.
* Create a Subnet named "CloudFoundry" in Virtual Network "boshvnet-crp".
* Create a network security group named "nsg-cf" in create resource group, and configure it if needed.

**Note**: the name of the resource can be changed, but you need to ensure the name of the resource is consistent with the cloud config.

---
## Connecting Virtual Network {: #connecting-vnets }

The VMs in one AZ need to be able to talk to VMs in the other AZ. You will need to connect them through a [VNet-to-VNet VPN gateway](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-howto-vnet-vnet-resource-manager-portal).

---
## Configure CPI and Cloud configs {: configuring-configs }

Now that the IaaS is configured, update your Director's [CPI config](cpi-config.md). Create a new file `cpi.yml` and copy following content to it, substitute variables enclosed by double brackets with your credentials.  If you use [Azure quickstart template](https://github.com/Azure/azure-quickstart-templates/tree/master/bosh-setup) creating your Director, the `ssh.public_key` can be found in `bosh-deployment-vars.yml`.

```yaml
cpis:
- name: azure-west-europe
  type: azure
  properties:
    environment: AzureCloud
    subscription_id: ((subscription_id))
    tenant_id: ((tenant_id))
    client_id: ((client_id))
    client_secret: ((client_secret))
    resource_group_name: ((resource_group_name))
    ssh_user: vcap
    ssh_public_key: ((ssh.public_key))
    default_security_group: ((default_security_group))
- name: azure-east-us-2
  type: azure
  properties:
    environment: AzureCloud
    subscription_id: ((subscription_id))
    tenant_id: ((tenant_id))
    client_id: ((client_id_2))
    client_secret: ((client_secret_2))
    resource_group_name: ((resource_group_name_2))
    ssh_user: vcap
    ssh_public_key: ((ssh.public_key))
    default_security_group: ((default_security_group_2))
```

```shell
$ bosh update-cpi-config cpi.yml
```

And use following content to replace `azs` and `networks` field in origin cloud config:

!!! note
    The `azs` section of your `cloud-config` now contains the `cpi` key with available values that are defined in your `cpi-config`.

```yaml
azs:
- name: z1
  cpi: azure-west-europe
  cloud_properties:
    availability_zone: '1'
- name: z2
  cpi: azure-east-us-2
  cloud_properties:
    availability_zone: '1'
- name: z3
  cpi: azure-west-europe
  cloud_properties:
    availability_zone: '2'

networks:
- name: default
  type: manual
  subnets:
  - azs:
    - z1
    - z3
    cloud_properties:
      security_group: nsg-cf
      subnet_name: CloudFoundry
      virtual_network_name: boshvnet-crp
    dns:
    - 8.8.8.8
    - 168.63.129.16
    gateway: 10.0.16.1
    range: 10.0.16.0/20
    reserved:
    - 10.0.16.0
    - 10.0.16.1
    - 10.0.16.2
    - 10.0.16.3
    - 10.0.16.255
  - azs:
    - z2
    cloud_properties:
      security_group: nsg-cf
      subnet_name: CloudFoundry
      virtual_network_name: boshvnet-crp
    dns:
    - 8.8.8.8
    - 168.63.129.16
    gateway: 10.1.16.1
    range: 10.1.16.0/20
    reserved:
    - 10.1.16.0
    - 10.1.16.1
    - 10.1.16.2
    - 10.1.16.3
    - 10.1.16.255
```

```shell
$ bosh update-cloud-config cloud.yml
```

Before deploying, perform the following actions:

- If VMs are associated with Azure Load Balancers, you need to put the VMs into zones `z1` and `z3`. For example, when deploying [cf-deployment](https://github.com/cloudfoundry/cf-deployment), you need to change [the azs of the router](https://github.com/cloudfoundry/cf-deployment/blob/master/cf-deployment.yml#L823-L826) to `z1` and `z3`. 
- In cloud config, change `load_balancer_name` to your newly created load balancer in previous step. If you are using Azure quickstart template and deploying with`deploy_cloud_foundry.sh`, you can simply change `-v load_balancer_name=cf-lb` to `-v load_balancer_name=<new_load_balancer_name>` in the script.
- Change system domain to <new_load_balancer_ip>.xip.io, or change it to a real domain that you possess and make the domain point to <new_load_balancer_ip>. If you are using Azure quickstart template and deploying with`deploy_cloud_foundry.sh`, you can simply change `-v system_domain=<old_load_balancer_ip>.xip.io` to `-v system_domain=<new_load_balancer_ip>.xip.io`.
- If you are using Azure quickstart template and deploying with`deploy_cloud_foundry.sh`, remove `-o ~/example_manifests/scale-to-one-az.yml` in the script.