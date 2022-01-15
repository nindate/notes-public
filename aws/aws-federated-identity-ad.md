### AWS Federated Identity with Microsoft Active Directory

1. [Install Server Roles](#install-server-roles) 
2. [Active Directory Configuration](#ad-config)
3. [Self Signed Certificate](#self-signed-cert)
4. [Active Directory Federated Services Configuration](#adfs-config)
5. [AWS Identity Provider and Roles](#aws-idp-roles)
6. [Access AWS Console using Federated Identity](#aws-access)


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

   * Give display name as Corporate ADFS

   * Select the service account - adfs_svc [give password]

   You need to Reboot server after above step completes

2. Download ADFS federation XML.  
   This will be required when creating Identity Provider in AWS.
   * Open Edge browser and paste the below URL to download the XML file  
   https://localhost/federationmetadata/2007-06/federationmetadata.xml

3. Configure Relying Party Trust in ADFS
   In Server Manager, open AD FS Management
   * Add Relying Party Trust
     * Click on Relying Party Trusts -> Add Relying Party Trust -> Claims aware
     * In the Import data about ... on a local network, paste followig URL for meatadata  
    https://signin.aws.amazon.com/static/saml-metadata.xml
     * Give display name as AWS Console Signin
     * Select Permit everyone
     * Save

   * Add Claims rules
     * Click the Relying Party Trust and select Edit Claim Issuance Policy
     * Add rule 1
     >* Claim rule template: Transform an incoming claim  
     >* rule name: NameId  
     >* Incoming claim type: Windows account name  
     >* Outgoing claim type: Name ID  
     >* Outgoing name ID format: Persistent Identifier
     >* Pass through all claim values: checked

     * Add rule 2
     >* Claim rule template: Send LDAP Attributes as Claims.
     >* rule name: RoleSessionName
     >* Attribute store: Active Directory
     >* LDAP Attribute: E-Mail-Addresses
     >* Outgoing Claim Type: https://aws.amazon.com/SAML/Attributes/RoleSessionName

     * Add rule 3
     >* Claim rule template: Send Claims Using a Custom Rule
     >* rule name: Get AD Groups
     >* Custom rule: 
     >```
     >c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
     >=> add(store = "Active Directory", types = ("http://temp/variable"), query = ";tokenGroups;{0}", param = c.Value);
     >```

     * Add rule 4
     >* Claim rule template: Send Claims Using a Custom Rule
     >* rule name: Roles
     >* Custom rule: 
     >```
     >c:[Type == "http://temp/variable", Value =~ "(?i)^AWS-([\d]{12})"] => issue(Type = "https://aws.amazon.com/SAML/Attributes/Role", Value = RegExReplace(c.Value, "AWS-([\d]{12})-", "arn:aws:iam::$1:saml-provider/CORP_AD,arn:aws:iam::$1:role/"));
     >```
     
     Few very important points to note over here. 
     * CORP_AD here refers to the name you will be configuring Identify Provider in AWS IAM. Thus if you will be using a different name, update this rule appropriately.
     * The above rule matches AD groups having names that start with AWS-any_twelve_digit_number- and maps them to AWS Role. e.g. matches AD group named AWS-12345789012-ADFS-EC2-ADMINS and maps to AWS role named ADFS-EC2-ADMINS
     * Thus any users in an AD group that matches above pattern will automatically get mapped to corresponding named AWS Role when they access AWS via the ADFS URL.

4. Open Powershell prompt and run following command.
   ```
   Set-AdfsProperties -EnableIdPInitiatedSignonPage $True
   ```
   Without the above configuration, when laundhing  the ADFS page it gives an error

5. Add support for Edge/Chrome/Mozilla browsers for single sign-on  
   By default only IE browser provides single sign-on via ADFS. To add Mozilla/Chrome browser or Edge browser to the list of supported browsers, carryout the following steps:
   * Add support for Mozilla/Chrome
   ```
   Set-AdfsProperties -WIASupportedUserAgents ((Get-ADFSProperties | Select -ExpandProperty WIASupportedUserAgents) + "Mozilla/5.0 (Windows NT")
   ```
   * Add support for Edge browser
   ```
   Set-AdfsProperties -WIASupportedUserAgents ((Get-ADFSProperties | Select -ExpandProperty WIASupportedUserAgents) + "Edge/12")
   ```


<a id='aws-idp-roles'></a>
### 5. AWS Identity Provider and Roles

1. Create Identity Provider in AWS of type SAML2.0.  
   You will need the ADFS federation XML that you downloaded in an earlier step.
  * Name it as CORP_AD (we have this name in one of the claim rules above)
  * Upload the ADFS federation XML

2. Create 2 Roles of type SAML 2.0 Federated. 
   * Select the CORP_AD as the identity provider, select API & Console access
   * Role named ADFS-EC2-ADMINS - having EC2 Full Access
   * Role named ADFS-S3-ADMINS - having S3 Full Access


<a id='aws-access'></a>
### 6. Access AWS Console using Federated Identity
If using IE browser, due to higher restrictions, you need to keep adding multiple sites to Trusted list of sites. Instead, better option is to use any of Edge/Chrome/Mozilla Firefox browser
1. Open an RDP session to the Windows server and login with one of the users - bob or mark
2. Open Edge browser or Chrome browser (you will need to install Chrome if you need). Launch the URL https://localhost/adfs/ls/idpinitiatedsignon.aspx
3. Select AWS Console Signin and signin. You will get automatically logged into AWS console (based on your AD login) and assigned appropriate role (as per AWS Role mapped with AD Group from the ADFS claims rule that was configured)



This is how you can use AWS Federated Identity with Microsoft AD and ADFS



