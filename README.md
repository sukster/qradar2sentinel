# Description
Send offense contributing events from IBM QRadar to Microsoft Sentinel and eventually correlate QRadar events with other events in Defender XDR. This solution is suitable for customers who want keep their QRadar SIEM but want to use Microsoft Sentinel (or Defender XDR) as a primary security console for incident investigations.

# Architecture
<img width="1533" height="1014" alt="image" src="https://github.com/user-attachments/assets/09796f37-c023-4d32-93db-5a3726e01743" />


# Deployment
The following deployment has been tested with QRadar ver. 7.5 UpdatePackage 11.

## Install On-premises Data Gateway
Let's start with installing the data gateway which will run on-premise where your QRadar is deployed. This data gateway will serve as a proxy between QRadar and Azure. It will allow the logic app to query the QRadar API and will ensure that all communication sent to Azure is encrypted.

## Install on Windows
On-premises Data Gateway supports either Windows Server or Windows 11. 


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

