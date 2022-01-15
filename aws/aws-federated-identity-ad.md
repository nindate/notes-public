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

#### Create new Forest
After all roles are installed, go to Notifications and click Promote server to domain server

Create a new forest and give it some name - e.g. mycompany.com

Server will automatically reboot after the above configuration is completed

#### Create Organizational Unit (OU) - optional
It is a good practise to create an OU to manage all relevant users and groups. This is optional however.

Create an Organizational Unit named AWS

#### Create Groups
Create Groups to group users that will have certain AWS accesses. 

Create following groups (replace 12345789012 with your AWS account number without any dashes)
* AWS-12345789012-ADFS-EC2-ADMINS - Users in this group will be provided EC2 Full access in AWS
* AWS-12345789012-ADFS-S3-ADMINS - Users in this group will be provided S3 Full access in AWS

#### Create Users
Create following login users, who will be provided necessary access to AWS
* bob
* mark

After the users are created, edit user bob and do the following:
* Set email address for bob e.g. bob@mycompamy.com
* Add user **bob** to following groups:
  * Administrators - to allow user to be able to use RDP (need to find better way to provide RDP access without adminstrator group membership)
  * AWS-12345789012-ADFS-EC2-ADMINS - to allow user to have EC2 Full access in AWS (after the ADFS and AWS side configurations are in place)

Edit user mark and do the following:
* Add user **mark** to following groups:
  * Administrators - to allow user to be able to use RDP (need to find better way to provide RDP access without adminstrator group membership)
  * AWS-12345789012-ADFS-S3-ADMINS - to allow user to have S3 Full access in AWS (after the ADFS and AWS side configurations are in place)


Create a service user named adfs_svc for ADFS. Set password never expires.

<a id='self-signed-cert'></a>
### 3. Self Signed Certificate


<a id='adfs-config'></a>
### 4. Active Directory Federated Services Configuration


<a id='aws-idp-roles'></a>
### 5. AWS Identity Provider and Roles