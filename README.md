# <p align=center>Install & Configure OpenShift Virtualization on Red Hat OCP </p>

In the following guide we will showcase how to install Red Hat OpenShift Container Platform using the Assisted Installer and install/configure Hypershift in Red Hat OpenShift Container Platform.The following operators will be installed as part of this installation:1) 

1) The following operators will be installed as part of this installation:
   1. MCE (Multi Cluster Engine)
   2. Local Storage
   3. ODF (OpenShift Data Foundation)
   4. OpenShift Virtualization 
   5. Configure the default Storage Class
2) Install and configure MetalLB Operator - Provides external IP to the workload domain API’s 
3) Configure the workload domains (using multicluster engine and OpenShift Virtualization), this provides the flexibility to have multiple hosted clusters, these hosted clusters will be deployed and configured as part of this guideIn continuation, the following tutorials will allow you to configure LDAP and RBACK with the Hub and the hosted clusters and also how to automate the deployment of hosted clusters using Red Hat OpenShift ACM:1) LDAP and Rbac

2) Automating hosted cluster deployment via Red Hat ACM **

# 1. Architecture

![](https://github.com/rohitralhan/hypershift/blob/main/images/architecture.png)

As shown in the diagram above we will be installing the Red Hat OpenShift Cluster (Hub cluster) on 6 physical machines. This will be a 3 master and 3 worker node custer. The hosted clusters worker nodes will be deployed on virtual machines, there will be 3 worker nodes each, deployed on a different virtual machine.Both the Hub and the hosted clusters will be integrated with an LDAP server with RBAC for both the Hub and the hosted. For the purposes of this demo we will be using Red Hat Identity Management as the LDAP server, you can also choose a different LDAP server if you wish.

# 2. Platform Requirements

The following will be required from a platform perspective in order to meet the test case needs:

* Subnet with DHCP enabled and properly configured
* 4 Static IP addresses on the same subnet and in the same CIDR as those served by DHCP
  - Two will be used by the “Management Domain”
  - Two will be used for the “Worker Domain”
* The cluster must consist of at least three OpenShift Container Platform worker or infrastructure nodes with locally attached-storage devices on each of them.
  - Each of the three selected nodes must have at least one raw block device available. OpenShift Data Foundation uses the one or more available raw block devices.
  - The devices you use must be empty, the disks must not include Physical Volumes (PVs), Volume Groups (VGs), or Logical Volumes (LVs) remaining on the disk.
  - Additional Notes
    - **_Make sure that disks used for ODF must not include Physical Volumes (PVs), Volume Groups (VGs), or Logical Volumes (LVs)._**
    - **_When selecting nodes for ODF select only worker nodes_**_._**

# 3. Install Red Hat OpenShift using Assisted Installer

This document will act as a high level guide for installing Red Hat OCP using Assisted Installer with a few operators required for setting up OpenShift Virtualization.To create the Red Hat OpenShift Cluster with the Assisted Installer web user interface, use the following procedure.
**Procedure**
1. Log in to the [Red Hat Hybrid Cloud Console](https://console.redhat.com/).
2. In the **Red Hat OpenShift** tile, click **Scale your applications**.
3. In the menu, click **Clusters**.
4. Click **Create cluster**.
5. Click the **Datacenter** tab.
6. Under **Assisted Installer**, click **Create cluster**.
7. Enter a name for the cluster in the **Cluster name** field.
8. Enter a base domain for the cluster in the **Base domain** field. All subdomains for the cluster will use this base domain.
9. Select the version of OpenShift Container Platform to install (4.14.13 in our case).\
   **_(_**_For_ **_custom network configuration_** _refer to the steps at the_ **_end_** _after_ **_bullet point 18_** _under section_ **_Custom Network Configuration (Static IP, Bridge and Bond))_**
10. Leave the other settings as is and click **Next** 
11. The Assisted Installer can install with certain Operators configured. The Operators include, please select all the three operators and click **Next**
    1. OpenShift Virtualization
    2. Multicluster engine (MCE) for Kubernetes
    3. Red Hat OpenShift Data Foundation
12. Next step is to add a host. Adding a host to the cluster involves generating a discovery ISO. The discovery ISO runs Red Hat Enterprise Linux CoreOS (RHCOS) in-memory with an agent.
    1. Click the **Add hosts** button and select the provisioning type.
       1. Select **Full image file: Provision with physical media** to download the larger full image. This is the recommended method for the `ppc64le` architecture and for the `s390x` architecture when installing with RHEL KVM.
       2. In the **SSH public key** field, click **Browse** to upload the `id_rsa.pub` file containing the SSH public key.
       3. Click **Generate Discovery ISO**.
       4. Next step is to download the **Discovery ISO** file and make it available as a boot media to all the servers on which the management domain (OpenShift Cluster) needs to be installed and boot from it. Close the popup. As the hosts start to boot from the ISO they will start appearing under the Host Inventory Section.
13. Once all the hosts are available and in Ready State click **Next.**
14. On the **Storage** screen click **Next**
15. In the **Networking** page, 
    1. select **Cluster-Managed Networking** for the **Network Management** and **OVN** for the **Network Type**
    2. For cluster-managed networking, configure the following settings:
       1. **Machine Network** Use the default network.
       2. **API virtual IP** make sure this IP is not used by any other device in the network.
       3. **Ingress virtual IP** make sure this IP is not used by any other device in the network.
       4. Click **Next**
16. Review the summary and click **Install Cluster**
17. On the Next screen you will be able to monitor the overall progress of the cluster and also for each and every node individually in the cluster.
18. Once the installation is complete you will be provided with the OpenShift Console URL along with the credentials. It will also provide the option to download the kubeconfig file, this file contains the cluster authentication information that can be used while executing commands via the openshift CLI. The animated image below shows installations steps mentioned above.
![](https://github.com/rohitralhan/hypershift/blob/main/images/AssistedInstaller/output.gif)

### **PS - Custom Network Configuration (Static IP, Bridge and Bond) Step 9 above**

Please refer to the steps below for configuring a custom network during the installation process. On the installer screen* 
**Go to Cluster Details** → **Host’s Network Configuration** → **Static IP, bridge and bonds** click **Next**
* Under **Security network configurations** → Select **YAML view** and enter the **below YAML** for each **host** in the cluster and provide the **MAC address** and the **Interface name** for each **host** and **NIC**
```yml
interfaces:
- name: bond0
  description: Bond
  type: bond
  state: up
  ipv4:
    enabled: true
    dhcp: true
  link-aggregation:
    mode: active-backup
    options:
      miimon: '100'
    port:
      - enp1s0
      - enp8s0

```
![](https://github.com/rohitralhan/hypershift/blob/main/images/BondSetUp/output.gif)

###

### Post Installation

When Install process shows complete
* Ensure you download and save the “**kubeconfig**” to your machine and name it “_kubeconfig-xxx_” or any other name of your choice, kubeconfig will be used to run **“oc”** commands as needed. Steps later in the document will be based on this name.

  ![](https://github.com/rohitralhan/hypershift/blob/main/images/download-kubeconfig.png)

* Log into the Web console for the cluster using the “**kubeadmin**” account credentials shown in the Installer page.
* Download and install the “**oc**” command. You will need this tool for later steps. Refer to the screenshot below on how to download the tools from the Red Hat OpenShift webconsole.
  - The “**oc**” tool is a binary which you can place at any location. Update the path variable with the location of the “**oc**” tool, this will allow you to run the “oc” tool from the CLI without specific the complete plath.

    ![](https://github.com/rohitralhan/hypershift/blob/main/images/download-oc2.png)
![](https://github.com/rohitralhan/hypershift/blob/main/images/download-oc1.png)

* Execut the “**oc login -u kubeadmin**” command provide the password when prompted for the kubeadmin user. You will need to use the “**oc**” cli tool later to execute commands, and thus must be logged in to execute those commands successfully.
* Follow the steps below to ensure the below operators are installed correctly
  - **Login** to the **Red OpenShift Console**, navigate to **Operators → Installed Operators,** validate the following operators are installed successfully
    - MCE (Multi Cluster Engine)
    - Local Storage
    - ODF
    - OpenShift Virtualization
![](https://github.com/rohitralhan/hypershift/blob/main/images/installed-operators.png)
* Enable the **OpenShift Data Foundation** operator **console** **plugin** by following the steps below:
  - In the Red Hat OpenShift Console navigate to **Operators → Installed Operators** click **OpenShift Data Foundation**
  - On the extreme right click under the **Console Plugin** (it should say Disabled/Enabled)
  - In the popup that opens select **Enabled** and click **Save**.
  - Click **Storage** in the left panel and you should see and option for **Data Foundation** as shown in the screenshot below.

    ![](https://github.com/rohitralhan/hypershift/blob/main/images/EnableConsolePlugin/output.gif)

# 4. Install and configure MetalLB Operator

MetalLB will be used as a provider to create externally accessible IP addresses. For this test scenario, this will be used to create the IP addresses that will be used by the Kubernetes API for the hosted clusters.MetalLB should be installed on the Hub cluster. Follow the instructions below to install the MetalLB operator.

## 4.1. Installation Procedure

1. In the OpenShift Container Platform web console, navigate to **Operators** → **OperatorHub**.
2. Type a keyword into the **Filter by keyword** box or scroll to find the Operator you want. For example, type metallb to find the MetalLB Operator.\
   You can also filter options by **Infrastructure Features**. For example, select **Disconnected** if you want to see Operators that work in disconnected environments, also known as restricted network environments.
3. On the **Install Operator** page, accept the defaults and click **Install**. Refer to the screenshots below.

![](https://github.com/rohitralhan/hypershift/blob/main/images/MetalLB/Install/output.gif)

## 4.2. Configuration Procedure

The steps below assume taking the simplest of IP Load Balancing configurations. There are multiple ways to configure MetalLB to work with your specific network such as **BGP**, and **BGP with BFD**, for this test, we will use the most universal configuration.
* Start by creating an instance of MetalLB

### 4.2.1. MetalLB Instance

1. Click **Operators** → **Installed Operators** → Click **MetalLB Operator**
2. Go to the **MetalLB Tab** and Click **Create MetalLB**
3. On the next screen accept the default values and click **Create**

![](https://github.com/rohitralhan/hypershift/blob/main/images/MetalLB/config/output4.gif)

### 4.2.2. Create IPAddressPool

Next create an IPAddressPool for use by MetalLB: [Configuring an address pool](https://docs.openshift.com/container-platform/4.14/networking/metallb/metallb-configure-address-pools.html#nw-metallb-configure-address-pool_configure-metallb-address-pools)

When configuring the IP Pool you should be sure to use the remaining two IP addresses you allocated earlier
1. Go to the **IPAddressPool Tab** and Click **Create IPAddressPool**
2. On the next screen provide the address pool and click **Create**

![](https://github.com/rohitralhan/hypershift/blob/main/images/MetalLB/config/output7.gif)

### 4.2.3. Create L2Advertising

Finally create an L2Advertisement using the IPAddress pool created in the prior step: [Configuring MetalLB with an L2 advertisement](https://docs.openshift.com/container-platform/4.14/networking/metallb/about-advertising-ipaddresspool.html#nw-metallb-configure-with-L2-advertisement_about-advertising-ip-address-pool)
1. Go to the **L2Advertisement Tab** and Click **Create L2Advertisement**
2. In the next step under the **ipAddressPools** select the **IPAddressPool** created in the previous step and click **Create**

![](https://github.com/rohitralhan/hypershift/blob/main/images/MetalLB/config/output10.gif)

# 5. MultiCluster-engine Operator for Kubernetes

MultiCluster-engine is used to create/manage the hosted clusters. MCE was installed as a part of the initial install, however we need to configure the OpenShift Router to handle wildcard routes.

## 5.1 Configure the cluster to Handle Wildcard Routes

In order to use the “Management Domain” OpenShift Router for application access we need to enable Wildcards. This command is listed below for your convenience. To execute the command you will need to RUN the **oc** command
```
$ export KUBECONFIG=kubeconfig
$ oc patch ingresscontroller -n openshift-ingress-operator default --type=json -p '[{ "op": "add", "path": "/spec/routeAdmission", "value": {wildcardPolicy: "WildcardsAllowed"}}]'
``` 

# 6. Deploy Hosted Clusters

The hosted clusters will be deployed as “Hosted Control Plane” clusters. This means that the control plane for the cluster exists in the hub cluster, and the Kubernetes worker nodes are deployed as virtual machines within OpenShift Virtualization. This method allows complete workload isolation from hosted cluster to hosted cluster.Deploying a Hosted Control Plane can be achieved in many ways. For this guide we will be using the “hcp” tool, and then later making additional changes to support LDAP authentication. **

## 6.1. Install the HCP Binary

You will need the “hcp” binary to create a hosted cluster, should be downloaded to the machine from where you will be connecting to the Red Hat OpenShift cluster via CLI, this could be a bastion host or another machine that can reach the OpenShift cluster. Follow the steps below to install the Workload Domain. **

### 6.1.1. Deploy worker for hosted cluster

You can name the cluster anything you want per documentation. Example below creates a cluster called “tenant1”
- You can download your OpenShift Container Platform pull secret file from [cloud.redhat.com/openshift/install/pull-secret](https://cloud.redhat.com/openshift/install/pull-secret) by selecting **Download pull secret**. Your OpenShift Container Platform pull secret is associated with your Red Hat Customer Portal ID, and is the same across all Kubernetes providers.
- It is suggested that you update the MEM, CPU and WORKER\_COUNT variables for this test as shown below:
```
$ export KUBECONFIG=kubeconfig-management
$ export CLUSTER_NAME=tenant1
$ export PULL_SECRET="$HOME/pull-secret.json"
$ export MEM="8Gi"
$ export CPU="4"
$ export WORKER_COUNT="3"
# run command to deploy cluster

$ hcp create cluster kubevirt   --name $CLUSTER_NAME   --node-pool-replicas $WORKER_COUNT   --pull-secret $PULL_SECRET   --memory $MEM   --cores $CPU --release-image=quay.io/openshift-release-dev/ocp-release:4.14.12-multi

or

alternatively , you can run the command by passing the values directly to the command as shown below

$ hcp create cluster kubevirt --name tenant1 --node-pool-replicas 3   --pull-secret $HOME/pull-secret.json --memory 8Gi --cores 4 --release-image=quay.io/openshift-release-dev/ocp-release:4.14.12-multi
```
- After executing the above command navigate to the Red Hat OpenShift web console and 
  - Navigate to **Home** → **Projects**
  - On the right had side you will see that there is a project created for each and every hosted cluster create with the nomenclature **clusters-<\<CLUSTER\_NAME as specified in the –name field>>** clusters-tenant1 in this case.

![](https://github.com/rohitralhan/hypershift/blob/main/images/projects.png)

- Click **clusters-tenant1** and scroll dow to see all the resources (pods, virtual machines etc) coming up.

![](https://github.com/rohitralhan/hypershift/blob/main/images/vms.png)

- When the install is complete be sure to download and save the kubeconfig file for this cluster and name it “_kubeconfig-xxx_”, we will need it later to executing commands via CLI on the hosted cluster.
  - Navigate to the Red Hat OpenShift Web Console and select All Clusters at the top. Here you will see all the hosted clusters listed

    ![](https://github.com/rohitralhan/hypershift/blob/main/images/hosted-clusters.png)
  - Click the hosted cluster of intereset (say tenant1) and in the screen that opens up you will see the Red hat OpenShift Web Console URL for the tenant1 cluster along with an option to download the kubeconfig file.

    ![](https://github.com/rohitralhan/hypershift/blob/main/images/console-url.png)

<BR><BR><BR>

### Click [here](https://github.com/rohitralhan/hypershift-ldap-rbac/README.md) to learn about setting up LDAP and Role based access control with the Hub and the hosted clusters
