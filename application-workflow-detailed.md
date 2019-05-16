# Detailed Application Authentication

This document outlines, in detail, the login and request process for an application using WebId-OIDC. In general, our user, *Alice* will be using a thrid-party application at `https://www.decentphotos.example`  to access data on both her pod at `https://alice.example` and her friend, Bob's pod at `https://bob.example`.

## Actors

In this example a multitude of actors are at play:

**Alice** - Alice will be providing consent for decentphotos to use her pod. Let's assume that Alice is using a standard web browser.

**Bob's Pod (RS)** - We will be trying to access photos Bob's Pod, known in the OIDC world as a Resource Server (RS). Bob is a friend of Alice. For this use case, let's assume that Bob has previously indicated via access control that Alice may access his photo using any app. You can read more about access control [here](https://github.com/solid/web-access-control-spec#referring-to-origins-ie-web-apps). For this example, bob's pod is at `bob.solid.example`.

**Alice's OP** - Alice's OpenID Provider (OP), also known as an IDP (Identity Provider), is the service responsible for authorizing our thrid-party app by providing it with the tokens necessary to gain access to any pod. In this demo, alice's OP is at `secureauth.example`.

**Alice's Pod (RS)** - Alice's Pod is hosted at `alice.coolpod.example`, giving Alice the webId of `https://alice.coolpod.example/profile/card#me`.

**Decent Photos (RP)** - decentphotos is a third party photo viewing application hosted at `https://www.decentphotos.example`. This app allows you to view your photos as well as your friend's photos. It will also perform cron jobs on the photos to detect faces. In the OIDC world this is known as the Relying Party (RP).

## Application Flow

![Application Login Flow](ApplicationLoginFlow.png)

### Authorization

Before any requests can be made, Alice must log in:

#### 1. Alice naviages to www.decentphotos.example

Alice has heard of a great new site that allows her to view her friend's photos and tag faces. She navigates to `www.decentphotos.example` via he web browser which returns and html page. This page contains JavaScript that will help with the authorization process.

#### 2. Alice clicks the "Connect" button

Before decentphotos can start displaying images, Alice needs to start the process of providing consent. To do so, she must either provider her webId (`https://alice.coolpod.example/profile/card#me`) or the service the url of her OP (`https://secureauth.example`) 

While it is not the case with Alice, a user's Pod and OP can be hosted at the same domain. For example, Bob's pod could be `bob.solid.example` with a webId of `https://bob.solid.example/profile/card#me`, but his OP is at `https://solid.example`.

#### 3. Request OP Configuration

Now that Alice has indicated either her webId or her OP's url, the RP must make a request to retrieve the OP's configuration.

If Alice entered her webId the request would be her webId's origin plus a path for the OIDC configuration:
```bash
GET https://alice.coolpod.example/.well-known/openid-configuration
```

If Alice entered her OP's url, the RP would simply append the OIDC configuration to the end.

```
GET https://secureauth.example/.well-known/openid-configuration
```

#### 4. Returns OP Configuration

Regardless of the what Alice entered, the response body should be the same. The [openid-configuration](https://auth0.com/docs/protocols/oidc/openid-connect-discovery) describes everything the client will need to know to authorize with Alice's specific OP.

Response Body:
```json
{  
   "issuer":"https://secureauth.example",
   "authorization_endpoint":"https://secureauth.example/authorize",
   "token_endpoint":"https://secureauth.example/token",
   "userinfo_endpoint":"https://secureauth.example/userinfo",
   "jwks_uri":"https://secureauth.example/jwks",
   "registration_endpoint":"https://secureauth.example/register",
   "response_types_supported":[  
      "code",
      "code token",
      "code id_token",
      "id_token",
      "id_token token",
      "code id_token token",
      "none"
   ],
   "response_modes_supported":[  
      "query",
      "fragment"
   ],
   "grant_types_supported":[  
      "authorization_code",
      "implicit",
      "refresh_token",
      "client_credentials"
   ],
   "subject_types_supported":[  
      "public"
   ],
   "id_token_signing_alg_values_supported":[  
      "RS256",
      "RS384",
      "RS512",
      "none"
   ],
   "token_endpoint_auth_methods_supported":[  
      "client_secret_basic"
   ],
   "token_endpoint_auth_signing_alg_values_supported":[  
      "RS256"
   ],
   "display_values_supported":[  

   ],
   "claim_types_supported":[  
      "normal"
   ],
   "claims_supported":[  

   ],
   "claims_parameter_supported":false,
   "request_parameter_supported":true,
   "request_uri_parameter_supported":false,
   "require_request_uri_registration":false,
   "check_session_iframe":"https://secureauth.example/session",
   "end_session_endpoint":"https://secureauth.example/logout"
}
```

*RECOMMENDATION: It is recommended that this configuration is saved to local storage so that it does not need to be retrieved every time the RP needs to make a request to the OP*

Currently in local storage:
```
OPENID_CONFIGURATION
```


#### 5. Generates a Private/Public key pair

WebId-OIDC depends on [Proof of Posession (PoP) tokens](README.md#securing-tokens-for-multiple-resource-servers). PoP tokens ensure that third-party applications can send requests to any number of Pods, while ensuring that evil pods can't steal a user's token.

The first step to generating a PoP token is generating a public and private key pair on the third-party RP. In our example, the private key is generated using `RSA256` and looks like:

```json
{  
   "alg":"RS256",
   "d":"UA69ET5dNnBYs1lbtNKEjozUXa3S5XYAjTkCcop1hxYAP4gM52iAkEQ0Jgs6CuVs74MxQM9vL__tMjH6KQ0sBvKTF_asnGYitftxBkABySwa-9bsDKtSrYO3F33Ctsiqp1WmZaT0eZi6rqjibm5ByYJRbyf9NDUI_farEaKoN6fTYGnIajNzzTKVm3PChbT4VyiOAADPGyJGOq5PX0oIbOHRG817NXYSZq8ZtDbm2_HoQwXjkSqz31d1UL8HKyghPWU4gBrkPyo_JWaoxbYlV5FGGXXUKBIvoaETf2-w5c7xNtpKJA7IS9IIKvZx-pVXP0nl5FjVC-I4ksQRY-FceQ",
   "dp":"Sa70sh4J65bbcDBEuHtx4wjxcsgDJjNHTbWc2B9HaZw49QF-sEI63SYezhF9DgR5oF_opEwjISAIv6bruHcb7Um7lxyxFV-iczR8NfO_wMhP9_EP8PyodBkX0YYQt2pBAizPwM_knKlRJOL3Dw5G2lnBDlzg6bWnA8oEM1UFvFk",
   "dq":"AmLtOjwwJPjt80yHGZc_OoFlKGDnm5m-5U8aS_kfIGbAFADYPuXacnMeOCVCthg8avCBgS68rC3hfNZeWkPnG2vXIiZqFLMPblQu6sjnlcvJS3peNQrJMs46ah6NJeUfLtniIcbveRu_CaosLwH_au5sPzpyStgELWiX5yV-bw",
   "e":"AQAB",
   "ext":true,
   "key_ops":[  
      "sign"
   ],
   "kty":"RSA",
   "n":"qpm6eDf1JvcOmhX1icr--xrD2_mx-SxpqIDArguhEcnhVsA6usKjttars25H8fpc_rtN6qvdhHCRHxIafLZ0PtWeZ9-CCKNgMnYViBW-F5j70RXSBfJI8zal2UrQQycbBhmPi6ATTLHaQyVfDM_rFpRmR99wh4h6QR2fBRAX2_J9_lPvFMofmjcYcfaOT9l3TIoghfW2ma3oLkIG4La0MSScfwzPFuAjqr7xO-sRGYB9OQDbOHJpuXOw0FYb8wxuMZjBRzFsudjaxlzlpr1eR6a5sTA8tAIs0f3j0oZQts358mMxp4oRPstUuExvrZcIQ7XODi3AvMwjbbqd5b1How",
   "p":"1nWyuXZpF7DcAXOxGzsod-itCTEI1hYv4CGXQ5daSvbasF0tqs9LuRzpTEGokLlzPINNjTPqo-lUElLIhdMkQsdbeYdIwI6FqmxjHx30V1yQqhhCsTT8RKRaHoN53EQsqb1r1RyEzGT6mhll7M0fMVxMuHnHTBPhrvMM5Yj6C_k",
   "q":"y6U0YyN1uFukE34iCh2bQDU4l1qpgHN_r1UzsSOzLKAo6vlqlLMdbzA5Fn5JHlDWZE7QkBSeTi845_O64MB1G6YvVIQTtGFgdom_p_DKO8pdFo3lIv9p3SRoHafBe2xb_bBT4wXAA71hzbydug95pJ0PN5CBw2-seZcalHcxf3s",
   "qi":"ZK7ILqxNETFmBOA3xiBZiT7pylUqUCjbnxc_Jv_1I8QyiCwO5WPjPuCeclZiPPVSf7IlA3mY9hB7HB72d33zIeL8Esd95iFj_rr4SiiQ9V5sS_vo1roAbfApTNx2uT7Jdcp5Td_362971qbvZbj8kvrrvgi7Dv7jffIhKx9ez4w"
}
```
From now on we will refer to this as `RP_PRIVATE_KEY`. The public key looks like:

```json
{  
  "jwk":{  
      "alg":"RS256",
      "e":"AQAB",
      "ext":true,
      "key_ops":[  
        "verify"
      ],
      "kty":"RSA",
      "n":"mVnn-HwEQi5mZR3Z0Wc7TBrJouOb7acKiseVSkjrj4mWCSEw21VTGNfovzUS71WYKoxFAd8zfkI9-lsAn3tkL8ppcMuI3F8KxsO86nNHSKrxZIlk-bP7RDFfpI9KWyifulKdipEmRit4iN-EFI2mK9KREscPWG083vqn4D81Xe4s0-gmRsBrVanwwu-mTwEKy8RFomV8CXOTcTNntdR3krluXZ38_uKBB1qg6_phBQwZ_sDMXWs8E90eCXhd_EQ6S8PGzCDPT2vg9wCB57ifAXt_8e4ZnqySmFPxegy7j3GcMuyhHzdpvv2fX5DvOxsgkjhsBzby9LD0bxStdBCSFQ"
  }
}
```
From now on we will refer to this as `RP_PUBLIC_KEY`.

#### 6. Saves the Public/Private key to local storage

The public/private key pair must be saved so that it can be referenced once the OP redirects back to the RP. It is recommended to save it to local storage, but that is not required.

Currently in local storage:
```
OPENID_CONFIGURATION
RP_PRIVATE_KEY
RP_PUBLIC_KEY
```

#### 7. Requests OP JWKs

Now that the RP's Public/Private keys are generated. The RP needs to be aware of the OP's public key. To do so a request should be made to the `jwks_uri` from the openid-configuration:

```
GET https://secureauth.example/jwks
```

#### 8. Returns OP JWKs

The [JSON Web Key Set (JWKS)](https://auth0.com/docs/jwks) is returned. These will eventually be used to validate the signature of the token the OP will eventually issue. It is recommended that the result of this request should be saved to local storage.

Response body:
```json
{  
  "keys":[  
    {  
      "kty":"RSA",
      "kid":"xeOjes9u3AcU4LBzcanEM7pZLwSlxaN7U62ZzOBDQuw",
      "alg":"RS256",
      "key_ops":[  
        "verify"
      ],
      "e":"AQAB",
      "n":"oB2LgkiZZ5iLAz1d4ua7sVxdbzY2nIRkDtf4UE08mWsD6UYRzLR98_gMAfnKB8i9yPCQkxfA5w_SZq6Y7odG1qSwLHM2mb_O2GSvY9kaG00UpeeEJCR19c7Jkcmq3GXh4yujnm2TFQ6YAzYNgrXkHlusaFUApJaQN6zr4AvmR_vX_5i__Ku7nuU-GbaV75LSr8o0QANdYFF0ooz5DJvydPplF8mO9_oD7ceSNLWP1AXlFs5JH6MEhH02dELb4-zeLcVzhoqON60cABTpbYSf1lLbYZsVUQ3cYE9CxXaByY2YNuQgc0k29mSmUvwEs0hNA5xUcE3-y_qKpYKniErb9Q"
    }
  ]
}
```

Currently in local storage:
```
OPENID_CONFIGURATION
RP_PRIVATE_KEY
RP_PUBLIC_KEY
OP_JWKS
```

#### 9. Sends Dynamic Registration Parameters

Now we have everything we need to perform [dynamic client registration](https://openid.net/specs/openid-connect-registration-1_0.html). Because each Solid user could have a different OP, it is not feasible to expect the developers of RPs to manually register with every OP. Therefore, the client (RP) is registered dynamically. A request is sent to the OP's `registration_endpoint`:

```
POST https://secureauth.example/register
```
Data:
```json
{
  grant_types: ["implicit"]
  issuer: "https://secureauth.example"
  redirect_uris: ["https://www.decentphotos.example/"]
  response_types: ["id_token token"]
  scope: "openid profile"
}
```

Each of these communicates something about the new client to the OP:
 - `grant_types`: A list of [OIDC grant types](http://docs.identityserver.io/en/latest/topics/grant_types.html) this client will use. `implicit` is great for web applications.
 - `issuer`: Alice's OP
 - `redirect_uris`: Redirect uris provided at the client registration stage state which redirect uris are valid during the authorization stage.
 - `response_types`: A list of [OIDC response types](https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html) the client can use. `id_token token` means that // TODO: Ask Dmitri about wrapping an id_token in a pop token. Should it not be the access token?
 - `scope`: OIDC uses scope as a way of defining what a client can have acces to. However, Solid has it's own access control system, so scope will always be `openid profile`

#### 10: Saves Client Information

The OP saves the client information to read later.

#### 11: Returns successful registration

The OP responds confirming the request, and with some new data.

Response Body:
```json
{  
  "client_id":"7243fd594bdcf9c71a9b902274afaa30",
  "redirect_uris":[  
    "https://chat.o.team/"
  ],
  "response_types":[  
    "id_token token"
  ],
  "grant_types":[  
    "implicit"
  ],
  "application_type":"web",
  "id_token_signed_response_alg":"RS256",
  "token_endpoint_auth_method":"client_secret_basic",
  "frontchannel_logout_session_required":false,
  "registration_access_token":"eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJodHRwczovL3NvbGlkLmNvbW11bml0eSIsInN1YiI6IjcyNDNmZDU5NGJkY2Y5YzcxYTliOTAyMjc0YWZhYTMwIiwiYXVkIjoiNzI0M2ZkNTk0YmRjZjljNzFhOWI5MDIyNzRhZmFhMzAifQ.gypiiq3K5_oRpEN7e1KaobI5CwWdrr7ZpwhdjUMneSEOEUVOwE0qeWlzu44j24eOIGeX8PzTc-f5ca0cRk22HSSRZIcAVo-GmGlZ4oAA5uGOuiEdncaF3ZKDv-0q1WXiyUFD_2hiiUrQwLtt-iSo3ZQOlCMP3opC6A73-b6UpOJTc9Q-V-sNTtOZ5IHpgOSkQZbgSuQr_vTGRLjoHl_v_8-AarjUIytbf_6h9iEFuPOyqugMUFALeNPBHSN8CTgJI3jcvx4HZtM9ByHNGGPjZv4TJrXy1sJIHokJkwdg1Pv_3mkUKVwtO4zxKAOu5MlspaZo6c-Oku__9S2mnu88xQ",
  "registration_client_uri":"https://solid.community/register/7243fd594bdcf9c71a9b902274afaa30",
  "client_id_issued_at":1557964995
}
```

The main one to keep note of is the `client_id` which will be used to tie subsequent requests back to this registered client.

This information should be saved to local storage.

Currently in local storage:
```
OPENID_CONFIGURATION
RP_PRIVATE_KEY
RP_PUBLIC_KEY
OP_JWKS
CLIENT_REGISTRATION_RESPONSE
```

#### 12. Authorization Request

// TODO complete explanation

#### 
