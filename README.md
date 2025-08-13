# Description
Send offense contributing events from IBM QRadar to Microsoft Sentinel and eventually correlate QRadar events with other events in Defender XDR.

# Architecture
<img width="1533" height="1014" alt="image" src="https://github.com/user-attachments/assets/09796f37-c023-4d32-93db-5a3726e01743" />


# Deployment

## Generate QRadar API token

### Create Security Profile
We will add all networks and all log sources assuming that the offense contributing events may come from various log sources and networks.
<img width="1075" height="638" alt="image" src="https://github.com/user-attachments/assets/3ad14851-65a7-4499-ba93-a7fbe4651855" />

