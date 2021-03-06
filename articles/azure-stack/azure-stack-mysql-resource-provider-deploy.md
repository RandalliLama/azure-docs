---
title: Use MySQL databases as PaaS on Azure Stack | Microsoft Docs
description: Learn how you can deploy the MySQL Resource Provider and provide MySQL databases as a service on Azure Stack.
services: azure-stack
documentationCenter: ''
author: mattbriggs
manager: bradleyb
editor: ''

ms.service: azure-stack
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 01/10/2018
ms.author: mabrigg

---

# Use MySQL databases on Microsoft Azure Stack

*Applies to: Azure Stack integrated systems and Azure Stack Development Kit*

You can deploy a MySQL resource provider on Azure Stack. After you deploy the resource provider, you can create MySQL servers and databases through Azure Resource Manager deployment templates. You can also provide MySQL databases as a service. 

MySQL databases, which are common on web sites, support many website platforms. For example, after you deploy the resource provider, you can create WordPress websites from the Web Apps platform as a service (PaaS) add-on for Azure Stack.

To deploy the MySQL provider on a system that does not have Internet access, copy the file [mysql-connector-net-6.10.5.msi](https://dev.mysql.com/get/Download/sConnector-Net/mysql-connector-net-6.10.5.msi) to a local share. Then provide that share name when you are prompted for it. You must install the Azure and Azure Stack PowerShell modules.


## MySQL Server resource provider adapter architecture

The resource provider is made up of three components:

- **The MySQL resource provider adapter VM**, which is a Windows virtual machine that's running the provider services.

- **The resource provider itself**, which processes provisioning requests and exposes database resources.

- **Servers that host MySQL Server**, which provide capacity for databases that are called hosting servers.

This release no longer creates MySQL instances. This means that you need to create them yourself and/or provide access to external SQL instances. Visit the [Azure Stack Quickstart Gallery](https://github.com/Azure/AzureStack-QuickStart-Templates/tree/master/mysql-standalone-server-windows) for an example template that can:
- Create a MySQL server for you.
- Download and deploy a MySQL Server from Azure Marketplace.

> [!NOTE]
> Hosting servers that are installed on a multi-node Azure Stack implementation must be created from a tenant subscription. They can't be created from the default provider subscription. They must be created from the tenant portal or from a PowerShell session with an appropriate sign-in. All hosting servers are chargeable VMs and must have appropriate licenses. The service administrator can be the owner of the tenant subscription.

### Required privileges
The system account must have the following privileges:

1.	Database: Create, drop
2.	Login: Create, set, drop, grant, revoke

## Deploy the resource provider

1. If you have not already done so, register your development kit and download the Windows Server 2016 Datacenter Core image downloadable through Marketplace management. You must use a Windows Server 2016 Core image. You can also use a script to create a [Windows Server 2016 image](https://docs.microsoft.com/azure/azure-stack/azure-stack-add-default-image). (Be sure to select the core option.) The .NET 3.5 runtime is no longer required.


2. Sign in to a host that can access the privileged endpoint VM.

    - On Azure SDK installations, sign in to the physical host. 
    - On multi-node systems, the host must be a system that can access the privileged endpoint.
    
    >[!NOTE]
    > The system on which the script is being run *must* be a Windows 10 or Windows Server 2016 system with the latest version of the .NET runtime installed. Installation fails otherwise. The Azure SDK host meets this criteria.
    

3. Download the MySQL resource provider binary. Then run the self-extractor to extract the contents to a temporary directory.

    >[!NOTE] 
    > The resource provider build corresponds to the Azure Stack builds. Be sure to download the correct binary for the version of Azure Stack that is running.

    | Azure Stack build | MySQL RP installer |
    | --- | --- |
    | 1.0.180102.3 or 1.0.180106.1 (multi-node) | [MySQL RP version 1.1.14.0](https://aka.ms/azurestackmysqlrp1712) |
    | 1.0.171122.1 | [MySQL RP version 1.1.12.0](https://aka.ms/azurestackmysqlrp1711) |
    | 1.0.171028.1 | [MySQL RP version 1.1.8.0](https://aka.ms/azurestackmysqlrp1710) |

4.  The Azure Stack root certificate is retrieved from the privileged endpoint. For the Azure SDK, a self-signed certificate is created as part of this process. For multi-node, you must provide an appropriate certificate.

    If you need to provide your own certificate, place a .pfx file in the **DependencyFilesLocalPath** that meets the following criteria:

    - Either a wildcard certificate for \*.dbadapter.\<region\>.\<external fqdn\> or a single site certificate with a common name of mysqladapter.dbadapter.\<region\>.\<external fqdn\>.

    - This certificate must be trusted. That is, the chain of trust must exist without requiring intermediate certificates.

    - Only a single certificate file exists in the DependencyFilesLocalPath.
    
    - The file name must not contain any special characters or spaces.


5. Open a **new** elevated (administrative) PowerShell console. Then change to the directory where you extracted the files. Use a new window to avoid problems that might arise from incorrect PowerShell modules that are already loaded on the system.

6. [Install Azure PowerShell version 1.2.11](azure-stack-powershell-install.md).

7. Run the `DeployMySqlProvider.ps1` script.

The script performs these steps:

* Downloads the MySQL connector binary (this can be provided offline).
* Uploads the certificates and other artifacts to a storage account on Azure Stack.
* Publishes gallery packages so that you can deploy SQL databases through the gallery.
* Publishes a gallery package for deploying hosting servers.
* Deploys a VM by using the Windows Server 2016 image that was created in step 1. It also installs the resource provider.
* Registers a local DNS record that maps to your resource provider VM.
* Registers your resource provider with the local Azure Resource Manager (tenant and admin).


You can:
- Specify the required parameters on the command line.
- Run without any parameters, and then enter them when prompted.

Here's an example you can run from the PowerShell prompt. Be sure to change the account information and passwords as needed:


```
# Install the AzureRM.Bootstrapper module, set the profile, and install the AzureRM and AzureStack modules.
Install-Module -Name AzureRm.BootStrapper -Force
Use-AzureRmProfile -Profile 2017-03-09-profile
Install-Module -Name AzureStack -RequiredVersion 1.2.11 -Force

# Use the NetBIOS name for the Azure Stack domain. On the Azure SDK, the default is AzureStack, and the default prefix is AzS.
# For integrated systems, the domain and the prefix are the same.
$domain = "AzureStack"
$prefix = "AzS"
$privilegedEndpoint = "$prefix-ERCS01"

# Point to the directory where the resource provider installation files were extracted.
$tempDir = 'C:\TEMP\MYSQLRP'

# The service admin account (can be Azure Active Directory or Active Directory Federation Services).
$serviceAdmin = "admin@mydomain.onmicrosoft.com"
$AdminPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force
$AdminCreds = New-Object System.Management.Automation.PSCredential ($serviceAdmin, $AdminPass)

# Set the credentials for the new resource provider VM.
$vmLocalAdminPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force
$vmLocalAdminCreds = New-Object System.Management.Automation.PSCredential ("mysqlrpadmin", $vmLocalAdminPass)

# And the cloudadmin credential required for privileged endpoint access.
$CloudAdminPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force
$CloudAdminCreds = New-Object System.Management.Automation.PSCredential ("$domain\cloudadmin", $CloudAdminPass)

# Change the following as appropriate.
$PfxPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force

# Run the installation script from the folder where you extracted the installation files.
# Find the ERCS01 IP address first, and make sure the certificate
# file is in the specified directory.
. $tempDir\DeployMySQLProvider.ps1 -AzCredential $AdminCreds `
  -VMLocalCredential $vmLocalAdminCreds `
  -CloudAdminCredential $cloudAdminCreds `
  -PrivilegedEndpoint $privilegedEndpoint `
  -DefaultSSLCertificatePassword $PfxPass `
  -DependencyFilesLocalPath $tempDir\cert `
  -AcceptLicense

 ```


### DeploySqlProvider.ps1 parameters
You can specify these parameters in the command line. If you do not, or if any parameter validation fails, you are prompted to provide the required parameters.

| Parameter name | Description | Comment or default value |
| --- | --- | --- |
| **CloudAdminCredential** | The credential for the cloud administrator, necessary for accessing the privileged endpoint. | _Required_ |
| **AzCredential** | The credentials for the Azure Stack service admin account. Use the same credentials that you used for deploying Azure Stack. | _Required_ |
| **VMLocalCredential** | The credentials for the local administrator account of the MySQL resource provider VM. | _Required_ |
| **PrivilegedEndpoint** | The IP address or DNS name of the privileged endpoint. |  _Required_ |
| **DependencyFilesLocalPath** | Path to a local share that contains [mysql-connector-net-6.10.5.msi](https://dev.mysql.com/get/Downloads/Connector-Net/mysql-connector-net-6.10.5.msi). If you provide one of these paths, the certificate file must be placed in this directory as well. | _Optional_ (_mandatory_ for multi-node) |
| **DefaultSSLCertificatePassword** | The password for the .pfx certificate. | _Required_ |
| **MaxRetryCount** | The number of times you want to retry each operation if there is a failure.| 2 |
| **RetryDuration** | The timeout interval between retries, in seconds. | 120 |
| **Uninstall** | Removes the resource provider and all associated resources (see the following notes). | No |
| **DebugMode** | Prevents automatic cleanup on failure. | No |
| **AcceptLicense** | Skips the prompt to accept the GPL license.  (http://www.gnu.org/licenses/old-licenses/gpl-2.0.html) | |



Depending on the system performance and download speeds, installation might take as little as 20 minutes or as long as several hours. If the **MySQLAdapter** blade is not available, refresh the admin portal.

> [!NOTE]
> If the installation takes more than 90 minutes, it might fail. If it does, you see a failure message on the screen and in the log file. The deployment is retried from the failing step. Systems that do not meet the recommended memory and core specifications might not be able to deploy the MySQL RP.



## Verify the deployment by using the Azure Stack portal

> [!NOTE]
>  After the installation script finishes running, you  need to refresh the portal to see the admin blade.


1. Sign in to the admin portal as the service administrator.

2. Verify that the deployment succeeded. Go to **Resource Groups**, and then select the **system.\<location\>.mysqladapter** resource group. Verify that all four deployments succeeded.

      ![Verify deployment of the MySQL RP](./media/azure-stack-mysql-rp-deploy/mysqlrp-verify.png)

## Provide capacity by connecting to a MySQL hosting server

1. Sign in to the Azure Stack portal as a service admin.

2. Select **ADMINISTRATIVE RESOURCES** > **MySQL Hosting Servers** > **+Add**.

	On the **MySQL Hosting Servers** blade, you can connect the MySQL Server resource provider to actual instances of MySQL Server that serve as the resource provider’s back end.

	![Hosting servers](./media/azure-stack-mysql-rp-deploy/mysql-add-hosting-server-2.png)

3. Provide the connection details of your MySQL Server instance. Be sure to provide the fully qualified domain name (FQDN) or a valid IPv4 address, and not the short VM name. This installation no longer provides a default MySQL instance. The size that's provided helps the resource provider manage the database capacity. It should be close to the physical capacity of the database server.

    > [!NOTE]
    >If the MySQL instance can be accessed by the tenant and admin Azure Resource Manager, it can be placed under control of the resource provider. The MySQL instance *must* be allocated exclusively to the resource provider.

4. As you add servers, you must assign them to a new or existing SKU to allow differentiation of service offerings.
  For example, you can have an enterprise instance providing:
    - Database capacity
    - Automatic backup
    - Reserve high-performance servers for individual departments
 

The SKU name should reflect the properties so that tenants can place their databases appropriately. All hosting servers in a SKU should have the same capabilities.

![Create a MySQL SKU](./media/azure-stack-mysql-rp-deploy/mysql-new-sku.png)


>[!NOTE]
> SKUs can take up to an hour to be visible in the portal. You cannot create a database until the SKU is created.


## Test your deployment by creating your first MySQL database


1. Sign in to the Azure Stack portal as a service admin.

2. Select **+ New** > **Data + Storage** > **MySQL Database**.

3. Provide the database details.

    ![Create a test MySQL database](./media/azure-stack-mysql-rp-deploy/mysql-create-db.png)

4. Select a SKU.

    ![Select a SKU](./media/azure-stack-mysql-rp-deploy/mysql-select-a-sku.png)

5. Create a login setting. You can reuse an existing login setting or create a new one. This setting contains the user name and password for the database.

    ![Create a new database login](./media/azure-stack-mysql-rp-deploy/create-new-login.png)

    The connections string includes the real database server name. Copy it from the portal.

    ![Get the connection string for the MySQL database](./media/azure-stack-mysql-rp-deploy/mysql-db-created.png)

> [!NOTE]
> The length of the user names cannot exceed 32 characters in MySQL 5.7. In earlier editions, it can't exceed 16 characters.


## Add capacity

Add capacity by adding additional MySQL servers in the Azure Stack portal. Additional servers can be added to a new or existing SKU. Make sure the server characteristics are the same.


## Make MySQL databases available to tenants
Create plans and offers to make MySQL databases available for tenants. For example, add the Microsoft.MySqlAdapter service, add a quota, and so on.

![Create plans and offers to include databases](./media/azure-stack-mysql-rp-deploy/mysql-new-plan.png)

## Update the administrative password
You can modify the password by first changing it on the MySQL server instance. Select **ADMINISTRATIVE RESOURCES** > **MySQL Hosting Servers**. Then select the hosting server. In the **Settings** panel, select **Password**.

![Update the admin password](./media/azure-stack-mysql-rp-deploy/mysql-update-password.png)

## Update the MySQL resource provider adapter (multi-node only, builds 1710 and later)
Whenever the Azure Stack build is updated, a new MySQL resource provider adapter is released. The existing adapter might continue to work. However, we recommend updating to the latest build as soon as possible after Azure Stack is updated. 

The update process is similar to the installation process that was described earlier. You create a new VM with the latest resource provider code. Then you migrate the settings to this new instance, including database and hosting server information. You also migrate the necessary DNS record.

Use the UpdateMySQLProvider.ps1 script with the same arguments that were described earlier. Provide the certificate here as well.

> [!NOTE]
> Update is only supported on multi-node systems.

```
# Install the AzureRM.Bootstrapper module, set the profile, and install AzureRM and AzureStack modules.
Install-Module -Name AzureRm.BootStrapper -Force
Use-AzureRmProfile -Profile 2017-03-09-profile
Install-Module -Name AzureStack -RequiredVersion 1.2.11 -Force

# Use the NetBIOS name for the Azure Stack domain. On the Azure SDK, the default is AzureStack and the default prefix is AzS.
# For integrated systems, the domain and the prefix are the same.
$domain = "AzureStack"
$prefix = "AzS"
$privilegedEndpoint = "$prefix-ERCS01"

# Point to the directory where the resource provider installation files were extracted.
$tempDir = 'C:\TEMP\SQLRP'

# The service admin account (can be Azure Active Directory or Active Directory Federation Services).
$serviceAdmin = "admin@mydomain.onmicrosoft.com"
$AdminPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force
$AdminCreds = New-Object System.Management.Automation.PSCredential ($serviceAdmin, $AdminPass)

# Set credentials for the new resource provider VM.
$vmLocalAdminPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force
$vmLocalAdminCreds = New-Object System.Management.Automation.PSCredential ("sqlrpadmin", $vmLocalAdminPass)

# And the cloudadmin credential required for privileged endpoint access.
$CloudAdminPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force
$CloudAdminCreds = New-Object System.Management.Automation.PSCredential ("$domain\cloudadmin", $CloudAdminPass)

# Change the following as appropriate.
$PfxPass = ConvertTo-SecureString "P@ssw0rd1" -AsPlainText -Force

# Change directory to the folder where you extracted the installation files.
# Then adjust the endpoints.
. $tempDir\UpdateMySQLProvider.ps1 -AzCredential $AdminCreds `
  -VMLocalCredential $vmLocalAdminCreds `
  -CloudAdminCredential $cloudAdminCreds `
  -PrivilegedEndpoint $privilegedEndpoint `
  -DefaultSSLCertificatePassword $PfxPass `
  -DependencyFilesLocalPath $tempDir\cert `
  -AcceptLicense
 ```

### UpdateMySQLProvider.ps1 parameters
You can specify these parameters in the command line. If you don't, or if any parameter validation fails, you are prompted to provide the required parameters.

| Parameter Name | Description | Comment or default value |
| --- | --- | --- |
| **CloudAdminCredential** | The credential for the cloud administrator, necessary for accessing the privileged endpoint. | _Required_ |
| **AzCredential** | The credentials for the Azure Stack service admin account. Use the same credentials as you used for deploying Azure Stack. | _Required_ |
| **VMLocalCredential** |The credentials for the local administrator account of the SQL resource provider VM. | _Required_ |
| **PrivilegedEndpoint** | The IP address or DNS name of the privileged endpoint. |  _Required_ |
| **DependencyFilesLocalPath** | Your certificate .pfx file must be placed in this directory as well. | _Optional_ (_mandatory_ for multi-node) |
| **DefaultSSLCertificatePassword** | The password for the .pfx certificate | _Required_ |
| **MaxRetryCount** | The number of times you want to retry each operation if there is a failure.| 2 |
| **RetryDuration** | The timeout interval between retries, in seconds. | 120 |
| **Uninstall** | Remove the resource provider and all associated resources (see the following notes). | No |
| **DebugMode** | Prevents automatic cleanup on failure. | No |
| **AcceptLicense** | Skips the prompt to accept the GPL license.  (http://www.gnu.org/licenses/old-licenses/gpl-2.0.html) | |

## Remove the MySQL resource provider adapter

To remove the resource provider, it is essential to first remove any dependencies.

1. Ensure that you have the original deployment package that you downloaded for this version of the resource provider.

2. All tenant databases must be deleted from the resource provider. (Deleting the tenant databases doesn't delete the data.) This task should be performed by the tenants themselves.

3. Tenants must unregister from the namespace.

4. The administrator must delete the hosting servers from the MySQL Adapter.

5. The administrator must delete any plans that reference the MySQL Adapter.

6. The administrator must delete any quotas that are associated with the MySQL Adapter.

7. Rerun the deployment script with the following elements:
    - The -Uninstall parameter
    - The Azure Resource Manager endpoints
    - The DirectoryTenantID
    - The credentials for the service administrator account
