# WebID-OIDC Detailed Application Workflow

## Actors
**Alice** - A user that wants to allow an app to access data in her pod  
**Alice.com** - Alice’s Solid Pod Provider. She is using it both as her *Identity Provider (OP)*, and as her *Resource Server (RS)* because her data is stored in her pod. This is the typical case for most Solid users.  
**DecentPhotos** - The decentralized app that Alice wants to use, making it the *Relying Party (RP)*. It is an autonomous agent that connects to a pod and organizes someones photos into different smart albums like - “photos taken in italy”, or “photos of justin"  

## Applicability
This approach can be employed whether the application is one that Alice uses directly on the web through a browser (like an application that lets her manage her photos), or one that runs autonomously. For example, an application that she has authorized to connect to her pod and use smarts to organize her photos for her.

## Assumptions
In this example, Alice is already logged into her identity provider (e.g. via solid-auth-client) and has her credentials readily available (i.e. in local storage). If she wasn’t, she would simply need to go through an extra challenge at Step #6, which would be specific to that identity provider and her preferences (i.e. username and password, two-factor, etc.)

## Step-by-Step Workflow
1. Alice hears about DecentPhotos and wants to give it access to her pod so it can organize her pictures for her.
2. DecentPhotos is an autonomous agent that runs on its own (from some cloud provider). Alice needs to give DecentPhotos some authority to access the photos in her pod on her behalf.
3. Alice goes to `www.decentphotos.com` to connect it to her pod. She enters her WebID so decent can lookup who her OP (Identity Provider) is, and clicks a button to connect.
4. decentphotos.com (RP) sends a redirect to the authorization endpoint at Alice.com because it is her OP, including in the request a client_id of the the decentphotos application/agent webid (`https://decentphotos.com/appid#this`)
5. Alice’s browser makes the request to the redirect url (which is the authorization endpoint at OP), identifying itself by the application webid (client_id), and also passing along an optional scope and the redirect_uri, which is a callback to decentphotos.com, to be used after alice has proved she has control of Alice.com (RS)
6. Because Alice is already logged in at her OP, she doesn’t need to enter her username and password again (she’s already got a token proving she is THE ALICE).
7. The authorization endpoint at Alice.com (OP) asks Alice if she wants to authorize DecentPhotos (RP) to access her Pod at a given scope. She has the ability here to further narrow this to only a subset of her photo library if she likes. Upon her confirmation here, DecentPhotos will be added as a trusted application in her WebID Profile, identified by its application/agent WebID (`https://decentphotos.com/appid#this`).
8. Alice submits this and is sent the redirect_uri / callback which was provided by DecentPhotos (RP) in Step #5, along with an authorization code, so her browser sends a get request to that with the code
9. DecentPhotos makes a request to the OP’s token endpoint with the authorization code, and receives a token in the response (assuming all checks out) like:  
```
  { access_token: “98qwerkjhqwerjhkwq”, token_type: “Bearer” }.
```
  The response nay also include additional details like scope / expiration time, or also a refresh token if available. The refresh token would allow DecentPhotos to continue to renew the bearer token without Alice having to intervene until the expire time associated with the Refresh token (which is typically set for weeks or months vs. a bearer which should be shorter lived (hours)).
10. DecentPhotos (RP) stores this token and is able to use it for requests to Alice.com (RS)
11. That night, while Alice is asleep, DecentPhotos (RP) is going to connect to Alice.com (RS) and organize some photos
12. DecentPhotos (RP) sends a request to the Alice.com (RS) for `https://alice.com/pics/private`, including the bearer token
13. Alice.com validates the bearer token against Alice.com (OP), and concludes it is valid. Since these are on the same server, this is fairly easy because the data is readily available. If they weren’t, this could be done through token introspection.
14. Alice.com (RS) has confirmed that this is in fact THE DecentPhotos, and moves on to checking to see if DecentPhotos is authorized to access `https://alice.com/pics/private` based on the associated ACL.
15. Alice.com (RS) matched the client_id in the token (`https://decentphotos.com/appid#this`), with a rule in the ACL which permits that the agent at `https://decentphotos.com/appid#this` has read and write access to `https://alice.com/pics/private`
16. The request is granted.
