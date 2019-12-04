# Crowdstrike API for Mac
This contains all the CrowdStrike API work I've done towards managing Crowdstrike Next Gen AntiVirus for the Mac.

We are currently using JAMF as an MDM, and Forescout as NAC.

***Note***: I am a complete beginner when it comes to scripting, proudly standing on the shoulders of others who have done the hard work for things like "how do I remove commas from that?" If you're an expert and see better/smarter ways to clean up my code, please feel free to share. I will happily take credit for your expertise.

HELPFUL TOOL: Postman. It's a lifesaver while you're trying to figure out things like Group IDs, Prevention Policy IDs, and other bits of necessary information that is not in the GUI for Crowdstrike. Of course, then you have to learn how to use Postman, but it's worth it. 

## PRIMER

First things first, you have to have an API client set up in Crowdstrike (see CS documentation)

Setting up your API Client will provide you with a Client ID and a Client Secret. Keep those handy; I will refer to them as `CLIENTID` & `CLIENTSECRE`T purely for sanitation purposes.

I've decided to document the individual steps in the hopes that it will help you understand the flow of things; if you're like me and just looking to grab some code, feel free to skip to whatever you need.

Crowdstrike API uses OAuth2 Tokens; you'll need to pull one every time you do an individual query- that's why Postman is nice- you can do a bunch of queries from Postman on the same token.

Some of the information, such as Group ID and Prevention Policy ID you only need to grab once and remember it for future reference. 

### To Pull the API Token
```zsh
curl -X POST "https://api.crowdstrike.com/oauth2/token" -H "accept: application/json" -H "Content-Type: application/x-www-form-urlencoded" -d "client_id=CLIENTID&client_secret=CLIENTSECRET" | grep "access_token" | awk '{print $2}' | sed 's/["",]//g'`
```
