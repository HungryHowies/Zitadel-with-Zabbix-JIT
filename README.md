# Zitadel-with-Zabbix-JIT-(Just-In-Time)


Overview:

This documentation shows How-To setup Zitadel and Zabbix for JIT (Just in Time). This is a basic demonstration of settings that are needed. More settings could be configured for a Production environment.

Prerequisite:
  * Latest Version of Zabbix Server with HTTPS
  * Zitadel-2.42.11 Self-Hosting with HTTPS
  * Ubuntu 22.0.4
  * Nginx Proxy
    
# Project and Application

Log into Zitadel instance.

Navigate to Organization ---> Projects.

Click "Create New Project" and set the project name, Click continue.

Under Application click "New".


There are two naming procedures.

1. Name the Application

2. Choose the SAML Application then name it (zabbix). Click continue.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/dd0b4fc2-2b6b-4b32-94e2-db4c2ab9024a)


Click the Application called Zabbix.


![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/224eab2c-4aa2-46da-88db-5e46109b60ea)

# Create a XML file.

NOTE: The key points for the XML file are:


* entityID="zabbix"
* oasis:names:tc:SAML:2.0:attrname-format:basic (NameIDFormat)
* Location="https://zabbix.domain.com/index.php" (SingleLogoutService)
* Location="https://zabbix.domain.com/index_sso.php?acs (AssertionConsumerService)

Open a text file and place the following in it, The example below replace domain.com with your Zabbix instance.

```
<?xml version="1.0"?>
<md:EntityDescriptor xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
                    entityID="zabbix">
    <md:SPSSODescriptor AuthnRequestsSigned="false" WantAssertionsSigned="true"
                    protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
<md:SingleLogoutService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
                    Location="https://zabbix.domain.com/index.php" />
<md:NameIDFormat>urn:oasis:names:tc:SAML:2.0:attrnameformat:
                    basic</md:NameIDFormat>
<md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTPPOST"
                    Location="https://zabbix.domain.com/index_sso.php?acs"
index="1" />
   </md:SPSSODescriptor>
</md:EntityDescriptor>
```

Save the file as xml.

Upload XML file for zabbix.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/34f3abb4-a366-4dc5-bb81-ef145bd59f14)

Click save.

Option #3 can be used but the XML file shown in the display needs the SingleLogoutService added.
Example:
```
<md:SingleLogoutService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
                    Location="https://zabbix.domain.com/index.php" />
```

Under the Zabbix Application, click the tic box called "Assert Roles on Authentication" and "Check
for Project on Authentication".


Using the end point /saml/v2/metadata/ and place it on the end of Zitadel server name.
Example: 

```
https://zitadel.com/saml/v2/metadata.
```
Copy the certificate from the /saml/v2/metadata/. In this example I used the third one from the top.

# Zabbix Setup

Once the certificate is copied from Zitadel, then place it in a file called idp.crt in zabbix-server directory /etc/zabbix/web/.

ensure the following configuration is in idp.crt file.

``` -----BEGIN CERTIFICATE----- and -----END CERTIFICATE------- ```

As shown below.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/7c8ba6c3-c913-4936-aed7-1164d8ef20a6)

Save the file.

Edit Zabbix file called zabbix.conf.php.

```
vi /etc/zabbix/web/zabbix.conf.php
```

I used certificates created from Lets encrypt (In my testing phase) for Zabbix php file. Bottom of the file,
uncomment the last 4 lines and add the full path to all three certificates.

All three certificates can be in the directory /etc/zabbix/web/. Any directory would work, so long as the full path is configured
and Zabbix has access to the certificates.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/c14a0faa-0e8f-48b8-8fa3-4c4c2703562b)

Configuration completed as shown below.

```
// Used for SAML authentication.
// Uncomment to override the default paths to SP private key, SP and IdP X.509 certificates, and to set
extra settings.
$SSO['SP_KEY'] = '/etc/zabbix/web/fullchain.key';
$SSO['SP_CERT'] = '/etc/zabbix/web/privkey.crt';
$SSO['IDP_CERT'] = '/etc/zabbix/web/idp.crt';
$SSO['SETTINGS'] = [];
```

# Zabbix Web UI

Login Zabbix  Web UI and navigate to Users --> Authentication.

The Authentication section, disable "Deprovisioned users group".

Click the "SAML settings",then enable following tic boxes.

```
Enable SAML authentication
Enable JIT provisioning
```

Configure the following settings section.

```
IdP entity ID: https://zitadel-build.domain.com/saml/v2/metadata
SSO service URL: https://zitadel-build.domain.com/saml/v2/SSO
SLO service URL: https://zitadel-build.domain.com/saml/v2/SLO
Username attribute: UserName
SP name ID format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic ( Matches Zitadel XML)
```

Click the tic box "Configure JIT provisioning".

Edit these settings as shown.

```
Group name attribute: groups
User name attribute: firstName
User last name attribute: lastName
```

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/0012bde0-3bb3-4ced-92d3-e4b9b124194f)

# User group mapping.

I used a wildcard in place of the User Group Mapping. At this point I havent found a way to add group/s to Zitadel to match this setting.

Click the Add button under "User group mapping". The following example of these settings are shown below.

```
SAML group pattern: *
User groups: Zabbix administrators
User role: Super admin role
```

Click "Add" to save settings.

Ensure the tic box is enabled for SCIM provisioning.

The full SAML settings configuration as shown below.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/65909247-2e64-49c2-9af7-c261e306ad75)



You should see the link to SSO.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/b13bdd51-d72a-438e-a566-7643581f3567)








