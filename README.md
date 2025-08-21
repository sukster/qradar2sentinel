# Description
Send offense contributing events from IBM QRadar to Microsoft Sentinel and eventually correlate QRadar events with other events in Defender XDR. This solution is suitable for customers who want keep their QRadar SIEM but want to use Microsoft Sentinel (or Defender XDR) as a primary security console for incident investigations.

# Architecture
<img width="1533" height="1014" alt="image" src="https://github.com/user-attachments/assets/09796f37-c023-4d32-93db-5a3726e01743" />
<br><br>

# How it works
The Logic App queries the QRadar API to fetch a list of open (or inactive) offenses. It will trigger AQL search to get contributing events for each offense. Then it will send each contributing event to the Sentinel's Log Analytics Workspace. Optionally, the Logic App can close the offense at the end. If you decide not to close the offenses, this is not an issue. The Logic App keeps track (using Reference Map) of which events it has already processed in the previous runs and it will only fetch any new contributing events. If there are no new contributing events for the offense for a specific period of time (default is 30 days) then tracking record in the Reference Map will automatically expire. If the offense is not manually closed during this period, the Logic App will fetch the all contributing events for the offense again which will result in event duplication in Sentinel. For this reason, you may want to adjust the 30-day period (in the Logic App -> Create ReferenceMap box) to a timeframe during which you are sure that the offense will be manually closed but you need to do this before you run the Logic App for the first time. If you already ran the Logic App and you want to update this time period, you will need to do this using QRadar API.

# Deployment
The following deployment has been tested with QRadar ver. 7.5 UpdatePackage 11. I recommend deploying all resources for this solution in the same Azure region.

## Install On-premises Data Gateway
Let's start with installing the data gateway which will run on-premise where your QRadar is deployed. This data gateway will serve as a proxy between QRadar and Azure. It will allow the logic app to query the QRadar API and will ensure that all communication sent to Azure is encrypted.

## Install Procedure on Windows
On-premises Data Gateway supports either Windows Server 2019 and higher or Windows 10 and higher. I recommend Windows Server for performace reasons but if you only have Windows 11 license, it should be fine. I successfully tested this deployement on Windows Server 2022 Standard which was Azure Arc joined and Windows 11 Enterprise which was Entra ID joined. From the networking perspective, make sure that your QRadar portal (and consequently its API) is accessible from this server - you can check that from the web browser on the server. <br>
Follow the installation procedure here: https://learn.microsoft.com/en-us/data-integration/gateway/service-gateway-install#download-and-install-a-standard-gateway. <br>
During the installation you will need to sign in with your organization's Microsoft 365 account. The gateway will be associated with that account. This could be a regular Microsoft 365 user account without additional administrative roles.<br>
Another important point to keep in mind during the gateway installation to install the gateway in the same Azure region where the logic apps custom connector will be deployed otherwise the gateway will not be able to use the custom connector. Changing the Azure region can be easily overlooked during the installation and so below is the screenshot where you can change it. 

<img width="920" height="633" alt="image" src="https://github.com/user-attachments/assets/965e1d2a-8566-4b6d-a8c2-253ec2eb99d9" />
<br>
Also make sure to save the gateway recovery key in a secure place just in case you need to reinstall the gateway to another server.
<br><br>

## Managing the Data Gateway
The M365 user who installed the gateway can manage the gateway at https://admin.powerplatform.microsoft.com/manage/ext/DataGateways. If you don't see the gateway make sure to select the correct region.
<br><br>
<img width="865" height="546" alt="image" src="https://github.com/user-attachments/assets/9cb37b8f-979e-481b-9f76-404938b0f227" />
<br><br>
It is highly recommended to assign a second gateway admin here. Click the gateway -> Manage users -> add a second admin user -> click the user and assign the Admin role.
<br><br>

## Securing the Gateway Server
Following the zero trust approach, configure the firewall rules on the Windows server to restrict communication from the server to other resources on your internal network except for QRadar API and possibly your DNS. If you can put the server into a zone behind a physical firewall this is even better.
<br><br>

## Deployment Procedure in Azure
We also need to create a On-premises data gateway resource in Azure. Follow the documentation here to deploy the gateway resource in Azure https://learn.microsoft.com/en-us/azure/logic-apps/connect-on-premises-data-sources?tabs=consumption. Important point to note is that to deploy the gateway in Azure you <strong>must use the same account</strong> that you used to sign in to Azure during the gateway installation on Windows. If this was a regular Microsoft 365 user account, make sure that this account has a contributor role to the Azure resource group (in our case "rg-sentinellogicapp").

<img width="877" height="704" alt="image" src="https://github.com/user-attachments/assets/ee035b48-b1c3-4d78-8ef6-3f115f96018c" />
<br><br>
Note: If you do not see the gateway name under the Installation Name, you need for wait for couple more minutes for it to show up.
<br><br>

## Generate QRadar API token

### Create Security Profile
We will create a new security profile called SentinelLogicApp and add all networks and all log sources assuming that the offense contributing events may come from various log sources and networks.
In QRadar go to Admin tab and select Security Profiles -> New

<img width="1075" height="638" alt="image" src="https://github.com/user-attachments/assets/3ad14851-65a7-4499-ba93-a7fbe4651855" />

### Create User Role
We will create a new user profile called SentinelLogicApp and select the following permissions. We will follow the least privileged principle.
In QRadar go to Admin tab and select User Roles -> New

<img width="1109" height="803" alt="image" src="https://github.com/user-attachments/assets/a7a88419-ff98-46ef-9799-2f4b494de819" />

Make sure you deploy the changes in QRadar Admin tab.

### Create Authorized Service
We will create a new authorized service and select the security profile and user profile we just created.
In QRadar go to Admin tab and select Authorized Services. Do not select to expire the authorized service.

<img width="1383" height="573" alt="image" src="https://github.com/user-attachments/assets/0cbc38ea-924a-42b5-94d4-545cddb3fa04" />

After the autorized service is created you will be given a security token. You need to copy this token to a secure location.

<img width="615" height="329" alt="image" src="https://github.com/user-attachments/assets/84832b7e-2897-42ae-b3f5-9131e19cd09a" />

## Create Azure Key Vault
We will store the QRadar API token safely in Azure Key Vault

### Create Resource Group
In Azure portal create a resource group under your subscription. We will store all resources for this solution in this resource group.

<img width="883" height="475" alt="image" src="https://github.com/user-attachments/assets/d81df188-f6a3-40d1-a39e-7420e1eda082" />
<br><br>

### Create Key Vault
In Azure portal and go to Key Vaults and click Create. Select the subscription and the resource group we just created. We will call the key vault "qradar-api".

<img width="909" height="830" alt="image" src="https://github.com/user-attachments/assets/77f5246e-4f52-475c-9de0-2961b90f9f0c" />


For the rest of the configuration settings on the key vault leave the defaults.

### Key Vault Secrets Officer role
So that you can add secrets to the key vault, you need to have the Key Vault Secrets Officer role. Add it to your account by in the qradar-api key vault by clicking Access Control (IAM) and Add Role Assignment.

<img width="1064" height="698" alt="image" src="https://github.com/user-attachments/assets/ca872c14-3046-4296-9363-c7c2c7974058" />


### Create Secret in Key Vault
We will add the QRadar API token to the key vault as a secret.
In Azure portal go to the qradar-api key vault and under Objects click Secrets. Then click Generate/Import button. We will call this secret "qradar-api-token". Paste the QRadar API token to the field called Secret value and click Create.

<img width="911" height="564" alt="image" src="https://github.com/user-attachments/assets/65f9f514-b10a-423d-b253-cc6746bc725d" />

## Deploy Logic Apps Custom Connector
We will deploy a custom connector for QRadar that will allow the logic app to query QRadar API using the data gateway. 

### Deployment in Azure
In Azure portal, create a new Logic Apps Custom Connector and call it "QRadar". 
<br><br>
<img width="876" height="653" alt="image" src="https://github.com/user-attachments/assets/fa95009a-48d7-4b57-a338-63bc84da1af8" />
<br><br>
Once the connector resource has been created, edit the connector and import the connector.json file which can be downloaded from this repository.

<img width="1253" height="618" alt="image" src="https://github.com/user-attachments/assets/4e842280-dc06-4efa-9bf0-6d92ad5e506a" />
<br><br>

Then in the General Information section, update the Host field and add the hostname (or FQDN) for your QRadar.  

<img width="797" height="851" alt="image" src="https://github.com/user-attachments/assets/78b4b58a-5e53-4702-b10b-c71b7f1fb183" />
<br><br>
This must be the same as the Common Name (CN) on your QRadar's SSL certificate.
<br><br>
<img width="1313" height="683" alt="image" src="https://github.com/user-attachments/assets/98720e11-bcdc-451b-bb03-76cee1fca663" />
<br><br>

### QRadar SSL Certificate considerations
Important point is that the <strong>data gateway does not support the default SSL certificate that comes with fresh QRadar installation. You must either generate a self-signed certificate or obtain a commercial certificate for your QRadar</strong>. In my case, I generated a self-signed certificate (see the QRadar self-signed certificate install procedure.txt in this repository) and then (because I don't have a DNS server in my lab) I used the hosts file on the server to ensure that it can resolve the FQDN to the QRadar IP address. Note that these certificate-related steps below may not be required in your case for example if you have a commercial certificate and DNS resolution.
<br><br>
<img width="1058" height="722" alt="image" src="https://github.com/user-attachments/assets/9976f46c-8b2f-4e74-b4f7-3622b0e2c47a" />
<br><br>

Finally, to make the certificate trusted by the Windows server, I had to export it (on the certificate there is an Export button) and import it to the Windows server's Trusted Root Certification Authorities store.
<br><br>
<img width="1247" height="713" alt="image" src="https://github.com/user-attachments/assets/f4bb87a7-4c15-4f5d-86f9-a59ed374c766" />
<br><br>
The easiest way is to download the certificate to the server, double-click it, select Install Certificate -> Local Machine -> select "Trusted Root Certification Authorities" store.
<br><br>

## Deploy Azure Logic App
Finally, we need to deploy and configure the Azure logic app.

### Deploy and Configure
You can deploy and configure the logic app to Azure by clicking the button below.
<br><br>
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsukster%2Fqradar2sentinel%2Frefs%2Fheads%2Fmain%2Fazuredeploy.json)

<br><br>
Configure the logic app template based on the screenshot below.
<br><br>
<img width="850" height="815" alt="image" src="https://github.com/user-attachments/assets/f8803976-8be8-48b7-8bc0-13db9f56770b" />
<br><br>
### Configure Permissions for Key Vault
The logic app needs to have a permission to read the secrets from the key vault. We configure this by granting the logic app the "Key Vault Secrets User" role on the qradar-api key vault.
<br><br>
<img width="947" height="622" alt="image" src="https://github.com/user-attachments/assets/11fb612a-d75d-409b-b7e1-227959c0abd4" />
<br><br>
Select the logic app name under managed identities.
<br><br>
<img width="750" height="528" alt="image" src="https://github.com/user-attachments/assets/830a014c-fc43-4d5b-ae65-c9da5e4db268" />
<br><br>
### Review Logic App in Editor
Open the logic app editor and check all the logic app boxes for any errors.
<br><br>
### Set Reoccurrence
Configure the first box in the logic app called Reoccurrence and set how often the logic app should execute.
<br><br>
### Enable the Logic App
The logic app is disabled by default so once everything has been configured you need to enable it to make it run.

## Offense Events in Sentinel
Check that the offense contributing events have been ingested into the OffenseEvents_CL table in the Sentinel workspace. Note that the event payload is Base64 encoded. We will need to decode it in the the Sentinel analytic rule.
<br><br>
<img width="882" height="467" alt="image" src="https://github.com/user-attachments/assets/2d2b9b23-a4fd-4dcf-ad6e-caa818e00083" />
<br><br>
## Create Analytic Rule in Sentinel
Although we have already ingested the offense events from QRadar to Sentinel, we may want to correlate the QRadar events with our existing events in Sentinel (or Defender XDR). For this reason, we need to create an analytic rule in Sentinel and map the Entities in the rule. Note that Defender XDR can automatically correlate the events from QRadar with existing events in Defender XDR but it will correlate based on the entities you map in the analytic rule.
<br><br>
Go to Sentinel and select Analytics -> Create -> Scheduled query rule.
<br><br>
<img width="619" height="602" alt="image" src="https://github.com/user-attachments/assets/5c5b949b-784a-4825-92fc-05eae205c8fc" />
<br><br>
Add the following query to the rule logic. The query will decode the Base64 payload and remove the original encoded field.
<br><br>
<strong>OffenseEvents_CL
| extend payload_decoded = base64_decode_tostring(payload_s)
| project-away payload_s</strong>
<br><br>
<img width="587" height="353" alt="image" src="https://github.com/user-attachments/assets/b4880cb9-9f27-4178-b2ce-cadcf0635f84" />
<br><br>
As mentioned, without entity mapping Defender XDR will not be able to automatically correlate its events with the new events from QRadar, therefore we need to create mappings for each entity that we want Defender XDR to correlate with. The following screenshot is just an idea. In practice, you may want to "extend" new entities from the payload but this will be specific to each use case and require testing and tuning.
<br><br>
<img width="557" height="374" alt="image" src="https://github.com/user-attachments/assets/d5d224c9-cf89-43ba-9609-4779fad0c1d7" />
<br><br>
For rest of the configuration of the analytic rule you may leave the default settings or modify them according to your requirements.
<br><br>

# SOC experience in Defender XDR
When the alerts generated by Sentinel (based on QRadar contributing events) and the alerts from Defender XDR contain the same entities (e.g. username), they will be automatically correlated by Defender XDR into a single incident.
<br><br>
<img width="1361" height="887" alt="image" src="https://github.com/user-attachments/assets/7166994c-1006-4167-99f2-ea5335639d40" />

