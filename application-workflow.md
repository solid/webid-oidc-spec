# WebID-OIDC Detailed Application Workflow

## Actors
**Alice** - A user that wants to allow an app access to data in her pod.  
**alice.example** - Alice’s Solid Pod Provider. She is using it both as her *Identity Provider (IdP)*, and as her *Resource Server (RS)* because her data is stored in her pod. This is the typical case for most Solid users.  
**DecentPhotos** - The decentralized app that Alice wants to use at decentphotos.example, making it the *Relying Party (RP)*. Once authorized, it will act as an autonomous agent that connects to Alice's pod on its own at a regular interval or through an event-based trigger, and will organize her photos into different smart albums like - “photos taken in Italy”, or “photos of Bob".

## Applicability
This approach can be employed whether the application is one that Alice uses directly on the web through a browser (like a website she visits that lets her manage her photos), or one that runs autonomously (like a service she has authorized to connect to her pod every night and use smarts to organize her photos for her).

## Assumptions
In this example, Alice is already logged into her identity provider (e.g. via solid-auth-client) and has her credentials readily available (i.e. in local storage). If she wasn’t, she would simply need to go through an extra challenge at Step #6, which would be specific to that identity provider and her preferences (i.e. username and password, two-factor, etc.)

## Step-by-Step Workflow
1. Alice hears about DecentPhotos and wants to give it access to her pod so it can organize her pictures for her.
2. DecentPhotos is an autonomous agent that runs on its own (from some cloud provider). To do this, Alice needs to give DecentPhotos some authority to access the photos in her pod on her behalf.
3. Alice goes to `www.decentphotos.example` to connect it to her pod. She enters her WebID so DecentPhotos can lookup who her IdP (Identity Provider) is, and clicks a 'Connect' button.
4. `decentphotos.example` (RP) reacts to the 'Connect' button being clicked by responding with a redirect. This redirect sends Alice's browser to the authorization endpoint at `alice.example` because it is her IdP, including in the request a client_id of the DecentPhotos application/agent WebID (`https://decentphotos.example/appid#this`).
5. Alice’s browser makes the request to the redirect URL (which is the authorization endpoint at IdP), identifying itself by the application WebID (client_id), and also passing along an optional scope and the redirect_uri, which is a callback to `decentphotos.example`, to be used after Alice has proved she has control of `alice.example` (RS)
6. Because Alice is already logged in at her IdP, she doesn’t need to enter her username and password again (she’s already got a token proving she is THE ALICE).
7. The authorization endpoint at `alice.example` (IdP) asks Alice if she wants to authorize DecentPhotos (RP) to access her Pod at a given scope. She has the ability here to further narrow this to only a subset of her photo library if she likes. Upon her confirmation here, DecentPhotos will be added as a trusted application in her WebID Profile, identified by its application/agent WebID (`https://decentphotos.example/appid#this`), and Alice's private photos folder at `https://alice.example/pics/private` will have its ACL updated to allow DecentPhotos (RP), identified by `https://decentphotos.example/appid#this`, read/write access to that folder and its contents.
8. Alice submits this and is sent (redirected) to the redirect_uri / callback which was provided by DecentPhotos (RP) in Step #5, along with an authorization code. Alice's browser now makes a new GET request to `decentphotos.example` (RP), with that authorization code included.
9. DecentPhotos makes a request to the IdP’s token endpoint with the authorization code, and receives a token in the response (assuming it all checks out) like:  
```
  { access_token: “98qwerkjhqwerjhkwq”, token_type: “Bearer” }.
```
  - The response may also include additional details like scope / expiration time, or a refresh token if available. The refresh token would allow DecentPhotos to continue to renew the bearer token without Alice having to intervene until the expire time associated with the Refresh token is passed. A refresh token's expire time is typically set for weeks or months, as opposed to the expire time on a bearer which should be much shorter lived (typically minutes or hours).

10. DecentPhotos (RP) stores this token and is able to use it for requests to `alice.example` (RS)
11. That night, while Alice is asleep, DecentPhotos (RP) is going to connect to `alice.example` (RS) and organize some photos
12. DecentPhotos (RP) sends a request to the `alice.example` (RS) for `https://alice.example/pics/private`, including the bearer token
13. `alice.example` (RS) validates the bearer token against `alice.example` (IdP), and concludes it is valid. Since these are on the same server, this is fairly easy because the data is readily available. If they weren’t, this could be done through token introspection.
14. `alice.example` (RS) has confirmed that this is in fact THE DecentPhotos, and moves on to checking to see if DecentPhotos is authorized to access `https://alice.example/pics/private` based on the associated ACL.
15. `alice.example` (RS) matched the client_id in the token (`https://decentphotos.example/appid#this`), with a rule in the ACL which permits that the agent at `https://decentphotos.example/appid#this` has read and write access to `https://alice.example/pics/private`
16. The request is granted.
