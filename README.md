# Description
Send offense contributing events from IBM QRadar to Microsoft Sentinel and eventually correlate QRadar events with other events in Defender XDR.

# Architecture
<img width="1533" height="1014" alt="image" src="https://github.com/user-attachments/assets/09796f37-c023-4d32-93db-5a3726e01743" />


# Deployment
The following deployment has been tested with QRadar ver. 7.5 UpdatePackage 11.


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
