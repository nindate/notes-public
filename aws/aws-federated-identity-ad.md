### AWS Federated Identity with Microsoft Active Directory

1. [Install Server Roles](#install-server-roles) 
2. [Active Directory Configuration](#ad-config)
3. [Self Signed Certificate](#self-signed-cert)
4. [Active Directory Federated Services Configuration](#adfs-config)
5. [AWS Identity Provider and Roles](#aws-idp-roles)


<a id='install-server-roles'></a>
### 1. Install Server Roles
Login to Windows server, launch Server Manager and install following server roles
* Active Directory Domain Services (ADDS)
* Active Directory Federated Services (ADFS)
* Domain Name Services (DNS)
* Internet Information Services (IIS)

Accept all default configurations for all roles.

<a id='ad-config'></a>
### 2. Active Directory Configuration

#### 2.1 Create new Forest
After all roles are installed, go to Notifications and click Promote server to domain server

Create a new forest and give it some name - e.g. mycompany.com

Server will automatically reboot after the above configuration is completed

#### 2.2 Create Organizational Unit (OU) - optional
It is a good practise to create an OU to manage all relevant users and groups. This is optional however.

Create an Organizational Unit named AWS

#### 2.3 Create Groups
Create Groups to group users that will have certain AWS accesses. 

Create following groups (replace 12345789012 with your AWS account number without any dashes)
* AWS-12345789012-ADFS-EC2-ADMINS - Users in this group will be provided EC2 Full access in AWS
* AWS-12345789012-ADFS-S3-ADMINS - Users in this group will be provided S3 Full access in AWS

#### 2.4 Create Users
1. Create following login users, who will be provided necessary access to AWS
   * bob
   * mark

2. Edit user bob and do the following:
   * Set email address for bob e.g. bob@mycompamy.com (This is important, because in ADFS rules it will be used)
   * Add user **bob** to following groups:
     * Administrators - to allow user to be able to use RDP (need to find better way to provide RDP access without adminstrator group membership)
     * AWS-12345789012-ADFS-EC2-ADMINS - to allow user to have EC2 Full access in AWS (after the ADFS and AWS side configurations are in place)

3. Edit user mark and do the following:
   * Set email address for mark e.g. mark@mycompamy.com (This is important, because in ADFS rules it will be used)
   * Add user **mark** to following groups:
     * Administrators - to allow user to be able to use RDP (need to find better way to provide RDP access without adminstrator group membership)
     * AWS-12345789012-ADFS-S3-ADMINS - to allow user to have S3 Full access in AWS (after the ADFS and AWS side configurations are in place)


4. Create a service account (user) named adfs_svc for ADFS. Set password never expires.

5. Set Service Principal Name (SPN) for the service account adfs_svc. Open Powershell prompt and run following command.

   ```setspn -a host/localhost adfs_svc```

<a id='self-signed-cert'></a>
### 3. Self Signed Certificate
Create a self signed certificate which will be required in the ADFS configuration.

From Server Manager, go to IIS Manager. 

Click on the IIS server name, from right side select Server certificates, then from right pane select Create Self-signed cert.
* Give it a name: ADFS-CERT
* Select Web hosting
* Set password

Once certificate is created, right-click the certificate and export certificate as ADFS_CERT.pfx [save to desktop]



<a id='adfs-config'></a>
### 4. Active Directory Federated Services Configuration
1. Create ADFS farm
   * In Server Manager, go to Notifications and click "Configure the federation service on this server"

   * Import Certificate

   * Give display name as AWS Console Signon

   * Select the service account - adfs_svc [give password]

   You need to Reboot server after above step completes

2. Download ADFS federation XML
   * Open Edge browser and paste the below URL to download the XML file
   https://localhost/federationmetadata/2007-06/federationmetadata.xml


<a id='aws-idp-roles'></a>
### 5. AWS Identity Provider and Roles