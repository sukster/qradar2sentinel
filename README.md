# Description
Send offense contributing events from IBM QRadar to Microsoft Sentinel and eventually correlate QRadar events with other events in Defender XDR. This solution is suitable for customers who want keep their QRadar SIEM but want to use Microsoft Sentinel (or Defender XDR) as a primary security console for incident investigations.

# Architecture
<img width="1533" height="1014" alt="image" src="https://github.com/user-attachments/assets/09796f37-c023-4d32-93db-5a3726e01743" />


# Deployment
The following deployment has been tested with QRadar ver. 7.5 UpdatePackage 11. I recommend deploying all resources for this solution in the same Azure region.

## Install On-premises Data Gateway
Let's start with installing the data gateway which will run on-premise where your QRadar is deployed. This data gateway will serve as a proxy between QRadar and Azure. It will allow the logic app to query the QRadar API and will ensure that all communication sent to Azure is encrypted.

## Install Procedure on Windows
On-premises Data Gateway supports either Windows Server 2019 and higher or Windows 10 and higher. I recommend Windows Server for performace reasons but if you only have Windows 11 license, it should be fine. I successfully tested this deployement on Windows Server 2022 Standard which was Azure Arc joined and Windows 11 Enterprise which was Entra ID joined. From the networking perspective, make sure that your QRadar portal (and consequently its API) is accessible from this server - you can check that from the web browser on the server. <br>
Follow the installation procedure here: https://learn.microsoft.com/en-us/data-integration/gateway/service-gateway-install#download-and-install-a-standard-gateway. <br>
During the installation you will need to sign in with your organization's Microsoft 365 account. The gateway will be associated with that account. This could be a regular Microsoft 365 user account without additional administrative roles however make sure that this account will not be disabled or deleted one day. Unfortunately, service principals are not supported yet.<br>
Another important point to keep in mind during the gateway installation to install the gateway in the same Azure region where the logic apps custom connector will be deployed otherwise the gateway will not be able to use the custom connector. Changing the Azure region can be easily overlooked during the installation and so below is the screenshot where you can change it. 

<img width="920" height="633" alt="image" src="https://github.com/user-attachments/assets/965e1d2a-8566-4b6d-a8c2-253ec2eb99d9" />

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
In Azure portal, create a new Logic Apps Custom Connector and call it "QRadar". 
<br><br>
<img width="876" height="653" alt="image" src="https://github.com/user-attachments/assets/fa95009a-48d7-4b57-a338-63bc84da1af8" />
<br><br>
Once the connector resource has been created, edit the connector and import the connector.json file which can be downloaded from this repository. Then click "Update connector".

<img width="1253" height="618" alt="image" src="https://github.com/user-attachments/assets/4e842280-dc06-4efa-9bf0-6d92ad5e506a" />

