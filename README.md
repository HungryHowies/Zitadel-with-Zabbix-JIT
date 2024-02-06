# Zitadel-with-Zabbix-JIT-(Just-In-Time)

Prerequisite:
  * Latest Version of Zabbix Server with HTTPS
  * Zitadel-2.42.11 Self-Hosting with HTTPS
  * Ubuntu 22.0.4
    
# Project and Application

Log into Zitadel instance.

Click "Create New Project" and call it Zabbix, Click continue.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/27e8b94a-6de5-403d-992f-d0fb5c786b1f)

Under Application click "New".

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/c9490992-c424-4fba-8a1e-38c40e9feae0)

Name it Zabbix then click SAML Application. Click continue.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/a1cdba83-7270-4072-aa8b-5deb10d4c4c5)

Click the Application called Zabbix.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/224eab2c-4aa2-46da-88db-5e46109b60ea)

# Create a XML file.

Open a text file and place the following in it,
NOTE: The key points for the XML file are:

```
entityID="zabbix"
oasis:names:tc:SAML:2.0:attrname-format:basic (NameIDFormat)
Location="https://zabbix.domain.com/index.php" (SingleLogoutService)
Location="https://zabbix.domain.com/index_sso.php?acs (AssertionConsumerService)
```

NOTE: Replace domain.com with your Zabbix instance.

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

The XML configuration should be shown below.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/cdea85f6-2d27-480b-b6e8-07036eb81979)


Click save.

Under the Zabbix Application, click the tic box called "Assert Roles on Authentication" and "Check
for Project on Authentication".

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/2fba50d0-980d-4f3c-bbbb-f206d49b62a7)

Using the end point /saml/v2/metadata/ and place it on the end of Zitadel server name.
Example: 

```
https://zitadel.com/saml/v2/metadata.
```
The certificate needed is  the third one from the top.

# Zabbix Setup

Copy the certificate from Zitadel and place it in a file called idp.crt in zabbix-server directory /etc/zabbix/web/.

Add the following  in idp.crt file.

-----BEGIN CERTIFICATE----- and -----END CERTIFICATE------- 

As shown below.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/7c8ba6c3-c913-4936-aed7-1164d8ef20a6)

Save the file.

Edit Zabbix file called zabbix.conf.php.

```
vi /etc/zabbix/web/zabbix.conf.php
```
I used certificates created from Lets encrypt (In my testing phase) for Zabbix php file. Bottom of the file,
uncomment the last 4 lines and add the full path to all three certificates.

All three certificates can be in the directory /etc/zabbix/web/ so long as the full path is configured
and Zabbix can access them, any directory would work.

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

Under Authentication section, disable "Deprovisioned users group".


![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/bfa6af08-dd48-4e0d-949d-7fbfd02b31a3)


Under the SAML settings, click the following tic boxes

```
Enable SAML authentication
Enable JIT provisioning
```
The following SAML Settings needs to be configured.
```
IdP entity ID: https://zitadel-build.domain.com/saml/v2/metadata
SSO service URL: https://zitadel-build.domain.com/saml/v2/SSO
SLO service URL: https://zitadel-build.domain.com/saml/v2/SLO
Username attribute: UserName
SP name ID format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic ( Matches Zitadel XML)
```
Click the tic box Configure JIT provisioning.
Edit these settings.
```
Group name attribute: groups
User name attribute: firstName
User last name attribute: lastName
```

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/0012bde0-3bb3-4ced-92d3-e4b9b124194f)

# User group mapping

I used a wildcard in place of the User Group Mapping.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/1b096739-2a58-489e-a2d1-805d5877572c)


Enable Enable SCIM provisioning

The full SAML settings configuration as shown below.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/b6a976a5-7e1f-40db-90a5-4eb7abec8dca)









