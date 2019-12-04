# Crowdstrike API for Mac
This contains all the CrowdStrike API work I've done towards managing Crowdstrike Next Gen AntiVirus for the Mac.

We are currently using JAMF as an MDM, and Forescout as NAC.

***Note***: I am a complete beginner when it comes to scripting, proudly standing on the shoulders of others who have done the hard work for things like "how do I remove commas from that?" If you're an expert and see better/smarter ways to clean up my code, please feel free to share. I will happily take credit for your expertise.

HELPFUL TOOL: Postman. It's a lifesaver while you're trying to figure out things like Group IDs, Prevention Policy IDs, and other bits of necessary information that is not in the GUI for Crowdstrike. Of course, then you have to learn how to use Postman, but it's worth it. 

## PRIMER

First things first, you have to have an API client set up in Crowdstrike (see CS documentation)

Setting up your API Client will provide you with a Client ID and a Client Secret. Keep those handy; I will refer to them as `CLIENTID` & `CLIENTSECRET` purely for sanitation purposes.

I've decided to document the individual steps in the hopes that it will help you understand the flow of things; if you're like me and just looking to grab some code, feel free to skip to whatever you need.

Crowdstrike API uses OAuth2 Tokens; you'll need to pull one every time you do an individual query- that's why Postman is nice- you can do a bunch of queries from Postman on the same token.

Some of the information, such as Group ID and Prevention Policy ID you only need to grab once and remember it for future reference. 

Create a Static Group in the Crowdstrike console that will hold the Macs.

Create the Prevention Policy (AV) in the Crowdstrike console and scope it to the Static Group.

***NOTE***: As of this writing, the Crowdstrike API can only add/subtract to static groups. 

#### To Pull the API Token
```zsh
curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
```

#### To Pull Group IDs
To assign a device to a group, you need the Group ID. The Group ID is not visible in the Crowdstrike Console, so it must be pulled via API.
The token must be requested first, and passed into a variable.
You will see the group(s) name, as well as "ids" above it; the ids is the Group ID
When you pull the Groups, it will also show you every device that's in that Group.

```
#Get the Token
Token=`curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
 
#Pass Token in and pull groups
curl -X GET "https://api.crowdstrike.com/devices/combined/host-groups/v1" -H "accept: application/json" -H "authorization: bearer $Token"
```
The return will be a list of both the Static and Dynamic Groups; have a look for your Static Group, and make a note of the ID. If you want, you could probably add grep to it;  I didn't in this example, in the off chance there will be a name change, or you want multiple group IDs for some reason.

#### Finding a Device ID
***NOTE*** 
"Device ID", "Sensor ID", and "Agent ID (AID)" are all the same thing to Crowdstrike. It is important that all dashes be removed, and that the ID be lower case, otherwise assigning it to a group won't work (see below).

```
sysctl cs.sensorid | awk '{print $2}' | awk '{print tolower($0)}' | sed -e 's/-//g'
```

#### Assigning a Mac to a Group
The script must pass the token and device ID into the request (The group IDs don't change, so I just add it in, rather than parsing through data that may change).
```
#Get the Token
Token=`curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
 
#Get the DeviceID aka the Agent ID aka the Sensor ID
DeviceID=`sysctl cs.sensorid | awk '{print $2}' | awk '{print tolower($0)}' | sed -e 's/-//g'`
 
#Move to Static Group
curl -X POST "https://api.crowdstrike.com/devices/entities/host-group-actions/v1?action_name=add-hosts" -H "accept: application/json" -H "Content-Type: application/json" -H "authorization: bearer $Token" -d "{ \"action_parameters\": [ { \"name\": \"filter\", \"value\": \"device_id:['$DeviceID']\" } ], \"ids\": [ \"INSERTGROUPIDHERE\" ]}"
```

#### Removing a Mac from a Group
This is very similar to adding, just a few small changes

```
#Get the OAuth2 Token
Token=`curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
 
#Find DeviceID
DeviceID=`sysctl cs.sensorid | awk '{print $2}' | awk '{print tolower($0)}' | sed -e 's/-//g'`
 
#Remove from AV group
curl -X POST "https://api.crowdstrike.com/devices/entities/host-group-actions/v1?action_name=remove-hosts" -H "accept: application/json" -H "Content-Type: application/json" -H "authorization: bearer $Token" -d "{ \"action_parameters\": [ { \"name\": \"filter\", \"value\": \"device_id:['$DeviceID']\" } ], \"ids\": [ \"INSERTGROUPIDHERE\" ]}"
```

### Checking to see if a Mac has a specific Prevention Policy applied to it
To see if a particular policy is being applied, most likely an AntiVirus (aka Prevention) policy, you go one of two routes:
-Check to see if it's in the correctly scoped group
-Check to see if a specific policy is listed in Device Info.
We went over checking Groups, so now let's check the device itself to see if a policy is applied

#### Find Policy IDs
Remember the Policy ID for future reference
```
#Get the Token
Token=`curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`

#Pass Token in and list all prevention policies
curl -X GET "https://api.crowdstrike.com/policy/combined/prevention/v1" -H "accept: application/json" -H "authorization: bearer $Token"
```

#### Find if a specific Policy ID is applied to a specific device
***NOTE*** The Policy ID is substituted with POLICYID
This script checks to see if a policy is applied to a device, returns *true* if it's applied, and *false* if it isn't.
```
#Get the OAuth2 Token
Token=`curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
 
#Find DeviceID
DeviceID=`sysctl cs.sensorid | awk '{print $2}' | awk '{print tolower($0)}' | sed -e 's/-//g'`
 
#Find AV Policy
AVpolicy=`curl -X GET "https://api.crowdstrike.com/devices/entities/devices/v1?ids=$DeviceID" -H "accept: application/json" -H "Content-Type: application/json" -H "authorization: bearer $Token" | grep POLICYID | awk '{print $2}' | sed -n '1p' | sed 's/["",]//g'`
 
#Mac AV Policy ID
PolicyID="POLICYID"
 
#Check if the found AV Policy matches the Mac AV Policy ID
if [[ "$AVpolicy" == "$PolicyID" ]]; then
echo "true"
else
echo "false"
 
fi
```

### Finding an individual Device
In the off chance you want to see everything applied to a device, and pull some other info:

```
Token=`curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
 
DeviceID=`sysctl cs.sensorid | awk '{print $2}' | awk '{print tolower($0)}' | sed -e 's/-//g'`
 
curl -X GET "https://api.crowdstrike.com/devices/entities/devices/v1?ids=$DeviceID" -H "accept: application/json" -H "authorization: bearer $Token"
```
