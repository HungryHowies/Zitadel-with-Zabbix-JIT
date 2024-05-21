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

Open a text file and place the following in it, The following steps are using a place holder called  **https://zabbix.domain.com**. 
The example below replace it by a FQDN or IP Address of the Zabbix-server.

```
<?xml version="1.0"?>
<md:EntityDescriptor xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata"
                     validUntil="2023-10-26T03:08:08Z"
                     cacheDuration="PT604800S"
                     entityID="https://zabbix.domain.com/">
    <md:SPSSODescriptor AuthnRequestsSigned="false" WantAssertionsSigned="false" protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
        <md:SingleLogoutService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect"
                                Location="https://zabbix.domain.com/index.php" />
        <md:NameIDFormat>urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified</md:NameIDFormat>
        <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
                                     Location="https://zabbix.domain.com/index_sso.php?acs" index="1" />
        
    </md:SPSSODescriptor>
</md:EntityDescriptor>
```

Save the file as xml.

Now upload XML file into Zitadel.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/34f3abb4-a366-4dc5-bb81-ef145bd59f14)

Click save.

Under the Zabbix Application, click the tic box called "Assert Roles on Authentication" and "Check
for Project on Authentication".

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/02891aa2-3b9c-42c3-a473-c7c924870d8c)


The SAML Metadata is located within the issuer domain. This would give us $CUSTOM-DOMAIN/saml/v2/metadata.


Example: 

```
https://zitadel.com/saml/v2/metadata.
```
Copy the certificate from the /saml/v2/metadata/. In this example I used the third one from the top. This will be used for Zabbix idp.crt file.

# Zabbix Setup

Once the certificate is copied from Zitadel, then place it in a file called idp.crt in zabbix-server directory /etc/zabbix/web/.

Example:

```
vi /etc/zabbix/web/idp.crt
```

Ensure the following configuration is in idp.crt file.

``` -----BEGIN CERTIFICATE----- and -----END CERTIFICATE------- ```



Once completed save the file and exit.


 
### Edit Zabbix file  zabbix.conf.php.

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


Edit the following file. This will prevent zabbix removing ACS endpoint during the session.

```
vi /usr/share/zabbix/vendor/onelogin/php-saml/src/Saml2/Utils.php
```
Change the following line from this…

```
private static $_proxyVars = false;
```
To this…

```
private static $_proxyVars = true;
```
Save and exit file. At this point  restart Zabbix service.

```
systemctl restart zabbix-server
```


# Zabbix Web UI

Login Zabbix Web UI and navigate to Users --> Authentication.

The Authentication section, disable "Deprovisioned users group".

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/73055e67-f88a-4ebd-bce4-f937aa0b5425)


Click the "SAML settings" tab,then enable following tic boxes.

```
Enable SAML authentication
Enable JIT provisioning
```

Configure the following settings section.

```
IdP entity ID: https://zitadel.domain.com/saml/v2/metadata
SSO service URL: https://zitadel.domain.com/saml/v2/SSO
SLO service URL: https://zitadel.domain.com/saml/v2/SLO
Username attribute: Email (I tried *UserName* but Zitadel does not have the users attribute)
SP name ID format: urn:oasis:names:tc:SAML:2.0:attrname-format:basic 
```

Click the tic box "Configure JIT provisioning".

Edit these settings as shown.

```
Group name attribute: zabbix
User name attribute: userName
User last name attribute: lastName
```
![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/d33ccaa0-8e17-4a58-91ae-5dd96619dbaf)

Ensure the tic box is enabled for SCIM provisioning.

# User group mapping.

In this section we are matching *Group name attribute* && *SAML group pattern*  with Zitadel users metadata.

*Group name attribute* is called zabbix. Under Zitadel users metadata this is called *key*

The *SAML group patterns* called admin (full access) and reader ( guest group & roles). Under Zitadel users metadata this is called Value.

Login to Zitadel and add users to the Project under AUTHORIZATIONS section.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/447bebfe-7001-4f21-bc25-ea8b9a3c2e1d)

Configure metadata for each user/s. 
Click on the user that has AUTHORIZATIONS for that project. On the left pane you should see Metadata section. This should match the configures in Goup mappings.
 
 Example:
 
 ![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/8db89bd0-eed8-49c6-9266-4162e3e2c4a0)

Click Save.

# Create a zitadel action.

Createa a Zitadel action to send the user metadata with the SAML responce.
Under the Action tab, navigate to the Script section and click the button called "New"
and apply this JavaScript. Copy and Paste.

```
/**
 * Add an custom attribute(s) to the SAMLResponse by pulling the value from 
 * metadata added to the user config.
 *
 * Flow: Complement SAMLResponse, Triggers: Pre SAMLResponse creation
 *
 * @param ctx
 * @param api
 */
function setCustomAttribute(ctx, api) {
    const user = ctx.v1.getUser()
    let metadata = ctx.v1.user.getMetadata()
        if (metadata === undefined || metadata.count == 0) {
        return
    }
    metadata.metadata.forEach(md => {
        api.v1.attributes.setCustomAttribute(md.key,'', md.value);
    })
}
```
Click Save.

Now create a Flow, this is below the Script section. The drop down, choose *Complement SAMLResponse* then click  *Add trigger* button.

Choose Trigger Type *Pre SAMLResponse creation*

Action, choose the script name created *setCustomAttribute*

Click save.

Example:

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/5162bc8b-b823-4749-85da-7c0eff3d3b84)



 




You should see the link to SSO.

![image](https://github.com/HungryHowies/Zitadel-with-Zabbix-JIT/assets/22652276/b13bdd51-d72a-438e-a566-7643581f3567)








