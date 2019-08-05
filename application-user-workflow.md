# Detailed Web Application Authentication

This document outlines, in detail, the login and request process for a web application using WebID-OIDC. In general, our user, *Alice* will be using a third-party web application at `https://www.decentphotos.example`  to access data on both her pod at `https://alice.example` and her friend, Bob's pod at `https://bob.example`.

## Actors

In this example a multitude of actors are at play:

**Alice** - Alice will be providing consent for decentphotos to use her pod. Let's assume that Alice is using a standard web browser.

**Bob's Pod (RS)** - We will be trying to access photos stored in Bob's Pod, known in the OIDC world as a Resource Server (RS). Bob is a friend of Alice. For this use case, let's assume that Bob has previously indicated via access control that Alice may access his photo using any web app. You can read more about access control [here](https://github.com/solid/web-access-control-spec#referring-to-origins-ie-web-apps). For this example, bob's pod is at `bob.solid.example`.

**Alice's OP** - Alice's OpenID Provider (OP), also known as an IDP (Identity Provider), is the service responsible for authorizing our third-party web app by providing it with the tokens necessary to gain access to any pod. In this demo, Alice's OP is at `secureauth.example`.

**Alice's Pod (RS)** - Alice's Pod is hosted at `alice.coolpod.example`, giving Alice the WebID of `https://alice.coolpod.example/profile/card#me`.

**Decent Photos (RP)** - decentphotos is a third party photo viewing web application hosted at `https://www.decentphotos.example`. This web app allows you to view your photos as well as your friend's photos. It will also perform cron jobs on the photos to detect faces. In the OIDC world, this is known as the Relying Party (RP).

## Application Flow

![Application Login Flow](ApplicationWorkflow.png)

### Authorization

Before any requests can be made, Alice must log in:

#### 1. Alice navigates to www.decentphotos.example

Alice has heard of a great new site that allows her to view her friend's photos and tag faces. She navigates to `www.decentphotos.example` via her web browser which returns an HTML page. This page contains JavaScript that will help with the authentication process.

#### 2. Alice clicks the "Connect" button

Before decentphotos can start displaying images, Alice needs to start the process of providing consent. To do so, she must either provide her WebID (`https://alice.coolpod.example/profile/card#me`) or the URL of her OP (`https://secureauth.example`) 

While it is not the case with Alice, a user's Pod and OP can be hosted at the same domain. For example, Bob's pod could be `bob.solid.example` with a WebID of `https://bob.solid.example/profile/card#me`, but his OP is at `https://solid.example`.

##### 2.1. Retrieve Profile

If Alice entered her Pod's URL rather than her OP's URL, a request should be made to determine her OP. There are two ways to do this and Pods may support one or both of them:

Firstly, the RP should make an `OPTIONS` request to the Pod:

```bash
OPTIONS https://alice.coolpod.example/profile/card#me
```

The Pod may return a link header similar to this:

```bash
RESPONSE HEADERS:
Link: <https://secureauth.example>; rel="http://openid.net/specs/connect/1.0/issuer", <card.acl>; rel="acl", <card.meta>; rel="describedBy", <http://www.w3.org/ns/ldp#Resource>; rel="type"
```

The attribute labeled by "http://openid.net/specs/connect/1.0/issuer" is the OP.

If the issuer is not in the Link header, make a `GET` request to the Pod.

```bash
GET https://alice.coolpod.example/profile/card#me
```

It will return a body similar to this:

```
@prefix : <#>.
@prefix solid: <http://www.w3.org/ns/solid/terms#>.
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
@prefix schema: <http://schema.org/>.

<>
    a foaf:PersonalProfileDocument ;
    foaf:maker <https://localhost:8443/profile/card#me> ;
    foaf:primaryTopic <https://localhost:8443/profile/card#me> .

:me a foaf:Person ;
    a schema:Person ;
    foaf:name "Alice" ;
    solid:oidcIssuer <https://secureauth.example> ;
```

The OP URL is located at `:me -> solid:oidcIssuer`

#### 3. Request OP Configuration

Now that we have Alice's OP's URL, the RP must make a request to retrieve the OP's configuration. This is always located at the OP's host followed by `/.well-known/openid-configuration`

```
GET https://secureauth.example/.well-known/openid-configuration
```

#### 4. Returns OP Configuration

Regardless of what Alice entered, the response body should be the same. The [openid-configuration](https://auth0.com/docs/protocols/oidc/openid-connect-discovery) describes everything the client will need to know to authorize with Alice's specific OP.

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

WebID-OIDC depends on [Proof of Possession (PoP) tokens](README.md#securing-tokens-for-multiple-resource-servers). PoP tokens ensure that third-party web applications can send requests to any number of Pods while ensuring that evil pods can't steal a user's token.

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
  "grant_types": ["implicit"],
  "issuer": "https://secureauth.example",
  "redirect_uris": ["https://www.decentphotos.example/"],
  "response_types": ["id_token token"],
  "scope": "openid profile"
}
```

Each of these communicates something about the new client to the OP:
 - `grant_types`: A list of [OIDC grant types](http://docs.identityserver.io/en/latest/topics/grant_types.html) this client will use. `implicit` is great for web applications.
 - `issuer`: Alice's OP
 - `redirect_uris`: Redirect URIs provided at the client registration stage state which redirect URIs are valid during the authorization stage.
 - `response_types`: A list of [OIDC response types](https://openid.net/specs/oauth-v2-multiple-response-types-1_0.html) the client can use. `id_token token` means that it should return both an Id Token (needed to identify this web application later) and an access token.
 - `scope`: OIDC uses 'scope' as a way of defining what a client can have access to. However, Solid has its own access control system, so scope will always be `openid profile`

#### 10: Saves Client Information

The OP saves the client information to read later.

#### 11: Returns successful registration

The OP responds confirming the request, and with some new data.

Response Body:
```json
{  
  "client_id":"7243fd594bdcf9c71a9b902274afaa30",
  "redirect_uris":[  
    "https://www.decentphotos.example/"
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
  "registration_client_uri":"https://secureauth.example/register/7243fd594bdcf9c71a9b902274afaa30",
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

Now that the web app is registered, we can finally make an auth request to authorize the web application.

```
GET https://secureauth.example/authorize?scope=openid&client_id=7243fd594bdcf9c71a9b902274afaa30&response_type=id_token%20token&redirect_uri=https%3A%2F%2Fwww.decentphotos.example%2F&request=eyJhbGciOiJub25lIn0.eyJyZWRpcmVjdF91cmkiOiJodHRwczovL2NoYXQuby50ZWFtLyIsImRpc3BsYXkiOiJwYWdlIiwibm9uY2UiOiJSWVRfUHFDMmpVMVNremoyTkViTjVFaUZFQ0szdTlad2dOcVlhVkl1RlFnIiwia2V5Ijp7ImFsZyI6IlJTMjU2IiwiZSI6IkFRQUIiLCJleHQiOnRydWUsImtleV9vcHMiOlsidmVyaWZ5Il0sImt0eSI6IlJTQSIsIm4iOiIyenltRTFQdDlPTm5saWpIMHlQN2ItZlUzZ0Vsb3liT3FNbVlKMHNTdFkxRU1DbzhUTXRLZkZtOXIwUnV6clRLMDJOX3VDNjZFcEdTaE9WeDJpMkRIMW1la1ZDeWV3a1pJM3BDZGVkUVk5dDhsY0VHUE5vcUVwSDViRkNZT2Jud3QtbHVUa0Q4N2hFX19MbWtrQV9wNG85MFZpREUwcEloYWxNaWt5SUsxNmZqV3I4dmFaeXRqbUVpUS04Z3k1SzZZSUpUdHJISXI4Yzd2dXprczJQeTlXb0xlOG8yVTBvaGFtRWd2QjhhbFJBUFFsUldiUHh3ZG5rVm85ZnI5UFBQelgzdHh4a2Q1NnN3andPZFJWalZwZmVoUVV0dDBuTHUwNDBjV3B2a1gwU1V4NnZMTklyYlExclM5UXkxMkdReGlqZDhfUjd1MFpIV1BoRkp3Y2hjancifX0.&state=L4F7Z7GrCu6cfxTh1qcEtCg1bfrs5daLhU5onmjeTS4
```

That URL might look a little complex, but it's essentially a request to `https://secureauth.example/authorize` with the following URL parameters:

 - `scope=openid`: a list of [OIDC scopes](https://auth0.com/docs/scopes/current/oidc-scopes) (attributes of the RS to which this token should have access). `open_id` is a scope that is needed to verify Alice's identity.
 - `client_id=7243fd594bdcf9c71a9b902274afaa30`: indicates the id of the client. The value for this field should be obtained in the registration phase.
 - `response_type=id_token%20token` indicates the desired response data. Note that you cannot use response types that were not previously indicated during registration.
 - `request=eyJhbGciOiJub25lIn0.eyJyZWRpc...`: A JWT containing the public key of the client and signed by the client using the private key. This is unique to WebID-OIDC. We will eventually use this to generate our pop-token.

 When decoded the request looks like

 ```json
{
  "redirect_uri": "https://www.decentphotos.example/",
  "display": "page",
  "nonce": "RYT_PqC2jU1Skzj2NEbN5EiFECK3u9ZwgNqYaVIuFQg",
  "key": {
    "alg": "RS256",
    "e": "AQAB",
    "ext": true,
    "key_ops": [
      "verify"
    ],
    "kty": "RSA",
    "n": "2zymE1Pt9ONnlijH0yP7b-fU3gEloybOqMmYJ0sStY1EMCo8TMtKfFm9r0RuzrTK02N_uC66EpGShOVx2i2DH1mekVCyewkZI3pCdedQY9t8lcEGPNoqEpH5bFCYObnwt-luTkD87hE__LmkkA_p4o90ViDE0pIhalMikyIK16fjWr8vaZytjmEiQ-8gy5K6YIJTtrHIr8c7vuzks2Py9WoLe8o2U0ohamEgvB8alRAPQlRWbPxwdnkVo9fr9PPPzX3txxkd56swjwOdRVjVpfehQUtt0nLu040cWpvkX0SUx6vLNIrbQ1rS9Qy12GQxijd8_R7u0ZHWPhFJwchcjw"
  }
}
 ```

 Notice that the `key` contains the public key generated earlier.

#### 13. Gets Alice's Consent

Given everything checks out in the last step, the OP should redirect to its login screen. The actual implementation of this is completely up to the OP. A user can log in with her password, a TLS certificate, or any other proven method of authentication. The important thing is that, thanks to the redirect, the control is now out of the hands of the RP and is in complete control of the OP.

#### 14. Generates an id_token

The OP should generate a JWT [id token](https://www.oauth.com/oauth2-servers/openid-connect/id-tokens/) as per the OIDC specifications. When decoded, the id_token could look like this.

```json
{
  "iss": "https://secureauth.example",
  "sub": "https://alice.coolpod.example/profile/card#me",
  "aud": "https://www.decentphotos.example",
  "exp": 1561685957,
  "iat": 1560476357,
  "jti": "dae3f18302519787",
  "nonce": "wy7zVjks3c6orHCVXs0K92sPpJQeUYehT6ioNkauZBE",
  "azp": "bcf8f216271b674be45a53a5a22d1f1f",
  "cnf": {
    "jwk": {
      "alg": "RS256",
      "e": "AQAB",
      "ext": true,
      "key_ops": [
        "verify"
      ],
      "kty": "RSA",
      "n": "zeBb4HV_zkoDRFXgNTqgfWI66Nj29YnFsu9yKdFTXCkfN6Vzj3mJbUj335iRqvp7bwJRuMhk-VYVL1paV2LMOllaqWfrf4LDh3sqysPnHBuL08Ejei3R9buoOaevHY-gyxcwm8vdLpeuWulN-t11vigDKLFzHdRxJRSby1-1D0KUo2K8L4yu0XHd2cUIb4ejC16NbzRBkTbKo4-HbNB0GhM36EMW_DvJJeANNcUFY_yVDFuIGUqQIvoE3wi581I9lLXak58jrRRf53Lx0nBOSpJIFL_0ljmUpLZIR4rdxTkTxXAzqHPmFUcwGhhDjiS2dQMFL5pjztOyqUxS-EyIhQ"
    }
  },
  "at_hash": "vbHexnl9TPOk3xYbGq-FFA"
}
```

There are a few important fields to note here:
 - `iss`: This issuer or OP that created and signed this token.
 - `sub`: The subject of the token, signifying that this token represents Alice.
 - `aud`: The audience of the token. This token is for the use of decentphotos and if it is provided to any other service that service should reject it because that means the token has been stolen.
 - `cnf`: This is the exact same public key passed to the OP before. It will later be used to help confirm that the OP is okay with pop tokens that have been signed by the RP.

#### 15. Redirects to redirect_url

Once everything is authenticated and the tokens are generated, the OP should redirect to the RP at the redirect URL. 

```
302 redirect to: https://www.decentpohtots.example/#access_token=eyJhbGciOiJSUzI1NiIsImtpZCI6Ik9pRHN6Q0xkR1c0In0.eyJpc3MiOiJodHRwczovL3NvbGlkLmNvbW11bml0eSIsInN1YiI6Imh0dHBzOi8vamFja3Nvbi5zb2xpZC5jb21tdW5pdHkvcHJvZmlsZS9jYXJkI21lIiwiYXVkIjoiYmNmOGYyMTYyNzFiNjc0YmU0NWE1M2E1YTIyZDFmMWYiLCJleHAiOjE1NjE2ODU5NTcsImlhdCI6MTU2MDQ3NjM1NywianRpIjoiYTU4ZGM4NzYwYmY3MWEwZCIsInNjb3BlIjoib3BlbmlkIn0.fhgHC-A_r_29VgHmI_U5VN7MPFozA1a2kChOFxZTlR70uXcDoj9UGEe5XDqAPHCl8vk-ZGNtN0DKk9z0pXaWM4PY5oDOEQTKsbwMOuB40E6IPDQoYp5Dgl1fvJG4rBUnOyd60PwXvRQKK2y1-f7iHVrnDA6DztZoGlr2HaN4s11mjdxhoAxw71J51osAqolPl2dm5TWh8Hu4ff-_UHkdlXR4baMkyOjmcWpfWzEyuxgQTWGfDvFIRQNCpbTMY172U1gCeINYJ4zBCqU7WgOGKeZoXVW4oJwp9cjpt2CsKOzFTIByLe61cKlLIClN17ahhWS2wTocz4jqMojylkFUHQ&token_type=Bearer&expires_in=1209600&id_token=eyJhbGciOiJSUzI1NiIsImtpZCI6Im02aGRrSnR5QUpNIn0.eyJpc3MiOiJodHRwczovL3NvbGlkLmNvbW11bml0eSIsInN1YiI6Imh0dHBzOi8vamFja3Nvbi5zb2xpZC5jb21tdW5pdHkvcHJvZmlsZS9jYXJkI21lIiwiYXVkIjoiYmNmOGYyMTYyNzFiNjc0YmU0NWE1M2E1YTIyZDFmMWYiLCJleHAiOjE1NjE2ODU5NTcsImlhdCI6MTU2MDQ3NjM1NywianRpIjoiZGFlM2YxODMwMjUxOTc4NyIsIm5vbmNlIjoid3k3elZqa3MzYzZvckhDVlhzMEs5MnNQcEpRZVVZZWhUNmlvTmthdVpCRSIsImF6cCI6ImJjZjhmMjE2MjcxYjY3NGJlNDVhNTNhNWEyMmQxZjFmIiwiY25mIjp7Imp3ayI6eyJhbGciOiJSUzI1NiIsImUiOiJBUUFCIiwiZXh0Ijp0cnVlLCJrZXlfb3BzIjpbInZlcmlmeSJdLCJrdHkiOiJSU0EiLCJuIjoiemVCYjRIVl96a29EUkZYZ05UcWdmV0k2Nk5qMjlZbkZzdTl5S2RGVFhDa2ZONlZ6ajNtSmJVajMzNWlScXZwN2J3SlJ1TWhrLVZZVkwxcGFWMkxNT2xsYXFXZnJmNExEaDNzcXlzUG5IQnVMMDhFamVpM1I5YnVvT2FldkhZLWd5eGN3bTh2ZExwZXVXdWxOLXQxMXZpZ0RLTEZ6SGRSeEpSU2J5MS0xRDBLVW8ySzhMNHl1MFhIZDJjVUliNGVqQzE2TmJ6UkJrVGJLbzQtSGJOQjBHaE0zNkVNV19EdkpKZUFOTmNVRllfeVZERnVJR1VxUUl2b0Uzd2k1ODFJOWxMWGFrNThqclJSZjUzTHgwbkJPU3BKSUZMXzBsam1VcExaSVI0cmR4VGtUeFhBenFIUG1GVWN3R2hoRGppUzJkUU1GTDVwanp0T3lxVXhTLUV5SWhRIn19LCJhdF9oYXNoIjoidmJIZXhubDlUUE9rM3hZYkdxLUZGQSJ9.Q5SfKAM4tnlYnFcpcagfd7xVi6U8T6Focax5MS6my36InYrMF7hfzy2611yAy_BpUittJj9nE5ONodoK-m-YERX1anfqfEjKMvQDKgKzZj5FsFT4sCLfIyHq-LcdVAsSz-y3e-x6nlLJVy3jiBs6vGUxbVRLciqE4zggSZx22GbJAWw457leMx9uvC2ijov3a8tR_ygn4ovHOgipfz8lkQUjL7AtOUqpon4oDTsOSVQBDELPJeh1AW3G3utV4HhPbwh2W6BFh3jUTmLZ9n80xxoachLqdKq6YIIrS38OjYNGKmQjb_KN5IiBrkufthG1l3rhamnxLxWkcPpGd0aKRg&state=kZdwwJSxUX_CkFYtqW5-50IaYMhNJUd5H2v69A8eOhE
```

#### 16. Saves the id_token to local storage

Now the RP has an id_token it can use to make requests. The login process is complete, and we can move on to making a request.

Currently in local storage:
```
OPENID_CONFIGURATION
RP_PRIVATE_KEY
RP_PUBLIC_KEY
OP_JWKS
CLIENT_REGISTRATION_RESPONSE
ID_TOKEN
```

### Sending a Request

#### 1. Wraps id_token in pop_token

A Solid web application could need to talk to any number of Resource Servers. Because of this, we do not want a single token that's used to talk to all of them. A malicious resource server could steal that token and pretend to be the web app. Instead, we wrap the id_token we've received from the OP in another token signed by the web app. This is called a pop_token.

When decoded, a pop_token can look like

```json
{
  "iss": "https://www.decentphotos.example",
  "aud": "https://bob.solid.example",
  "exp": 1560481857,
  "iat": 1560478257,
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6Im02aGRrSnR5QUpNIn0.eyJpc3MiOiJodHRwczovL3NvbGlkLmNvbW11bml0eSIsInN1YiI6Imh0dHBzOi8vamFja3Nvbi5zb2xpZC5jb21tdW5pdHkvcHJvZmlsZS9jYXJkI21lIiwiYXVkIjoiYmNmOGYyMTYyNzFiNjc0YmU0NWE1M2E1YTIyZDFmMWYiLCJleHAiOjE1NjE2ODU5NTcsImlhdCI6MTU2MDQ3NjM1NywianRpIjoiZGFlM2YxODMwMjUxOTc4NyIsIm5vbmNlIjoid3k3elZqa3MzYzZvckhDVlhzMEs5MnNQcEpRZVVZZWhUNmlvTmthdVpCRSIsImF6cCI6ImJjZjhmMjE2MjcxYjY3NGJlNDVhNTNhNWEyMmQxZjFmIiwiY25mIjp7Imp3ayI6eyJhbGciOiJSUzI1NiIsImUiOiJBUUFCIiwiZXh0Ijp0cnVlLCJrZXlfb3BzIjpbInZlcmlmeSJdLCJrdHkiOiJSU0EiLCJuIjoiemVCYjRIVl96a29EUkZYZ05UcWdmV0k2Nk5qMjlZbkZzdTl5S2RGVFhDa2ZONlZ6ajNtSmJVajMzNWlScXZwN2J3SlJ1TWhrLVZZVkwxcGFWMkxNT2xsYXFXZnJmNExEaDNzcXlzUG5IQnVMMDhFamVpM1I5YnVvT2FldkhZLWd5eGN3bTh2ZExwZXVXdWxOLXQxMXZpZ0RLTEZ6SGRSeEpSU2J5MS0xRDBLVW8ySzhMNHl1MFhIZDJjVUliNGVqQzE2TmJ6UkJrVGJLbzQtSGJOQjBHaE0zNkVNV19EdkpKZUFOTmNVRllfeVZERnVJR1VxUUl2b0Uzd2k1ODFJOWxMWGFrNThqclJSZjUzTHgwbkJPU3BKSUZMXzBsam1VcExaSVI0cmR4VGtUeFhBenFIUG1GVWN3R2hoRGppUzJkUU1GTDVwanp0T3lxVXhTLUV5SWhRIn19LCJhdF9oYXNoIjoidmJIZXhubDlUUE9rM3hZYkdxLUZGQSJ9.Q5SfKAM4tnlYnFcpcagfd7xVi6U8T6Focax5MS6my36InYrMF7hfzy2611yAy_BpUittJj9nE5ONodoK-m-YERX1anfqfEjKMvQDKgKzZj5FsFT4sCLfIyHq-LcdVAsSz-y3e-x6nlLJVy3jiBs6vGUxbVRLciqE4zggSZx22GbJAWw457leMx9uvC2ijov3a8tR_ygn4ovHOgipfz8lkQUjL7AtOUqpon4oDTsOSVQBDELPJeh1AW3G3utV4HhPbwh2W6BFh3jUTmLZ9n80xxoachLqdKq6YIIrS38OjYNGKmQjb_KN5IiBrkufthG1l3rhamnxLxWkcPpGd0aKRg",
  "token_type": "pop"
}
```

Notice this in this new token, the web app is now the issuer and the audience is the specific server to which we are sending a request. The id_token field contains the same token that we've saved to local storage.

#### 2. Request sent

When you send a request, the pop_token MUST be included as an `authorization` header as a bearer token.

```
GET https://bob.solid.example/photos/my_photos_with_alice/vacation.png
HEADERS:
  authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJiY2Y4ZjIxNjI3MWI2NzRiZTQ1YTUzYTVhMjJkMWYxZiIsImF1ZCI6Imh0dHBzOi8vamFja3Nvbi5zb2xpZC5jb21tdW5pdHkiLCJleHAiOjE1NjA0ODE4NTcsImlhdCI6MTU2MDQ3ODI1NywiaWRfdG9rZW4iOiJleUpoYkdjaU9pSlNVekkxTmlJc0ltdHBaQ0k2SW0wMmFHUnJTblI1UVVwTkluMC5leUpwYzNNaU9pSm9kSFJ3Y3pvdkwzTnZiR2xrTG1OdmJXMTFibWwwZVNJc0luTjFZaUk2SW1oMGRIQnpPaTh2YW1GamEzTnZiaTV6YjJ4cFpDNWpiMjF0ZFc1cGRIa3ZjSEp2Wm1sc1pTOWpZWEprSTIxbElpd2lZWFZrSWpvaVltTm1PR1l5TVRZeU56RmlOamMwWW1VME5XRTFNMkUxWVRJeVpERm1NV1lpTENKbGVIQWlPakUxTmpFMk9EVTVOVGNzSW1saGRDSTZNVFUyTURRM05qTTFOeXdpYW5ScElqb2laR0ZsTTJZeE9ETXdNalV4T1RjNE55SXNJbTV2Ym1ObElqb2lkM2szZWxacWEzTXpZelp2Y2toRFZsaHpNRXM1TW5OUWNFcFJaVlZaWldoVU5tbHZUbXRoZFZwQ1JTSXNJbUY2Y0NJNkltSmpaamhtTWpFMk1qY3hZalkzTkdKbE5EVmhOVE5oTldFeU1tUXhaakZtSWl3aVkyNW1JanA3SW1wM2F5STZleUpoYkdjaU9pSlNVekkxTmlJc0ltVWlPaUpCVVVGQ0lpd2laWGgwSWpwMGNuVmxMQ0pyWlhsZmIzQnpJanBiSW5abGNtbG1lU0pkTENKcmRIa2lPaUpTVTBFaUxDSnVJam9pZW1WQ1lqUklWbDk2YTI5RVVrWllaMDVVY1dkbVYwazJOazVxTWpsWmJrWnpkVGw1UzJSR1ZGaERhMlpPTmxaNmFqTnRTbUpWYWpNek5XbFNjWFp3TjJKM1NsSjFUV2hyTFZaWlZrd3hjR0ZXTWt4TlQyeHNZWEZYWm5KbU5FeEVhRE56Y1hselVHNUlRblZNTURoRmFtVnBNMUk1WW5WdlQyRmxka2haTFdkNWVHTjNiVGgyWkV4d1pYVlhkV3hPTFhReE1YWnBaMFJMVEVaNlNHUlNlRXBTVTJKNU1TMHhSREJMVlc4eVN6aE1OSGwxTUZoSVpESmpWVWxpTkdWcVF6RTJUbUo2VWtKclZHSkxielF0U0dKT1FqQkhhRTB6TmtWTlYxOUVka3BLWlVGT1RtTlZSbGxmZVZaRVJuVkpSMVZ4VVVsMmIwVXpkMmsxT0RGSk9XeE1XR0ZyTlRocWNsSlNaalV6VEhnd2JrSlBVM0JLU1VaTVh6QnNhbTFWY0V4YVNWSTBjbVI0Vkd0VWVGaEJlbkZJVUcxR1ZXTjNSMmhvUkdwcFV6SmtVVTFHVERWd2FucDBUM2x4VlhoVExVVjVTV2hSSW4xOUxDSmhkRjlvWVhOb0lqb2lkbUpJWlhodWJEbFVVRTlyTTNoWllrZHhMVVpHUVNKOS5RNVNmS0FNNHRubFluRmNwY2FnZmQ3eFZpNlU4VDZGb2NheDVNUzZteTM2SW5Zck1GN2hmenkyNjExeUF5X0JwVWl0dEpqOW5FNU9Ob2RvSy1tLVlFUlgxYW5mcWZFaktNdlFES2dLelpqNUZzRlQ0c0NMZkl5SHEtTGNkVkFzU3oteTNlLXg2bmxMSlZ5M2ppQnM2dkdVeGJWUkxjaXFFNHpnZ1NaeDIyR2JKQVd3NDU3bGVNeDl1dkMyaWpvdjNhOHRSX3lnbjRvdkhPZ2lwZno4bGtRVWpMN0F0T1VxcG9uNG9EVHNPU1ZRQkRFTFBKZWgxQVczRzN1dFY0SGhQYndoMlc2QkZoM2pVVG1MWjluODB4eG9hY2hMcWRLcTZZSUlyUzM4T2pZTkdLbVFqYl9LTjVJaUJya3VmdGhHMWwzcmhhbW54THhXa2NQcEdkMGFLUmciLCJ0b2tlbl90eXBlIjoicG9wIn0.sZFJEcnhZibDwZ1NlXtR7FaPW0e2w6VCym9Xo0d1vHA8SscARLM3oFB1a9io3XFxxT2iABnuaZZL7XM7IdsaxNCPz2sqgrhFQNpQxvLygjmLvv7tZz25hvNjgBz6RbWtL2XjfpW3LLZDH0_-Xj1aVMleunDjN9JP1J43EDEgyAUs0yB8ewdYHENQJBgnnmVRaGOUjSibwsevK_aBQzIFL8zEkel-c-s1iLYhjfrEdL77he2Tyy5lRRgmNDCXDx0DyDc_WleXirSV3kaDcXh49tCPDauak0lScZl1A61CsKnOL9gps-W00QLeQo6vwzlf33zw2QYo05RxBd4fS82IQQ
```

#### 3. Checks Audience

When an RS receives a pop_token it should first to see if the audience refers to itself. If it does not it MUST reject the request with a 403.

#### 4. Checks client signature.

The RS wants to ensure that this pop_token truly came from the client that the OP said it should come from when it signed the id_token. To do so, the RS should ensure that the public key included in the `cnf` field of the id_token matches the signature of the pop_token. If they do not match the RS MUST reject the request with a 403.

#### 5. Retrieves Profile

A request should be made to URL in the `sub` field of the id_token (The user's Pod) to determine that user's OP via the issuer attribute. For instructions on this see step 2.1 of the authorization instructions.

#### 6. Checks Issuer

Compare the issuer retrieved in step 5 with the issuer of the id token. If they are not identical the RS must reject the request with a 403.

#### 7. Retrieves OP Configuration

Before the RS requests information from the third-party OP, it must retrieve its configuration. The OP's address can be obtained via the `iss` field of the id_token (https://secureauth.example). Recall how to retrieve the OP's configuration in steps 3 and 4 of the authorization instructions.

```
GET https://secureauth.example/.well-known/openid-configuration
```

#### 8. Requests public keys

Now that we've confirmed the validity of the pop_token and the subject of the identity token, we want to ensure the validity of the id_token it contains. To do so, we need the OP's public keys. The route to retrieve the public keys is listed in the OP Configuration fetched in the previous step. Recall how to retrieve the OP's public keys in steps 7, and 8 of the authorization instructions.

```
GET https://secureauth.example/jwks
```

In order to decrease network usage, RSs SHOULD cache these keys once received and skip this step given keys are already in the cache.

#### 9. Performs Authentication

With OP public keys obtained, the RS can confirm if the id_token was signed by the OP. If it was not, the RS must reject the request with a 403.

#### 10. Returns Result

Given all went well, the RS should return the requested content.