## Motivation for WebID-OIDC

The Solid team, while [researching additional authentication
mechanisms](https://github.com/solid/solid/issues/22) to run alongside the
existing one
([WebID-TLS](https://github.com/solid/solid-spec/blob/master/authn-webid-tls.md)),
performed a review of existing authentication protocols that has been used by
(or proposed for) decentralized application ecosystems. Several criteria were
kept in mind.

One, we wanted to avoid the common pitfalls that are often encountered
when dealing with authentication for decentralized systems: the use of HTTP
Basic or Digest Auth, and relying on Bearer Tokens as an authentication
mechanism.

Secondly, we were looking for a protocol that did not rely solely on identifying
users via public/private crypto keys. Aside from the general difficulty people
have with storing and managing crypto keys, public client applications (browser
side Javascript apps, for example) do not have access to private key storage
facilities or keychain APIs. (And even the upcoming
[WebAuthn](https://www.w3.org/TR/webauthn/) spec requires the user to *register*
an account for each origin/domain -- precisely the situation that most
decentralized social app projects like Solid are trying to fix.) This ruled out
proposals such as HTTP Signatures, most "Identity on the Blockchain" proposals
including Bitcoin, and other client-side key management based systems.

If at all possible, we were looking for protocols that used standards and tech
stacks that were familiar and comprehensible to app developers (which ruled out
the older XML-based SAML protocol, as well as earlier incarnations of OpenID).

Other properties that we were looking for in a protocol included:

* Support for the full range of app and agent types, including browser side
  Javascript apps, traditional server-side web apps, mobile apps and desktop
  apps, and so on. The ability to support authentication of non-browser-based
  agents (such as server-side feed readers, desktop and mobile apps, etc) meant
  that we needed alternatives to the "Authenticate to your pod with a username
  and password and rely on WebID-TLS delegation through the pod" strategy.
* Addressed and incorporated security best practices and recommendations (see,
  for example [RFC 4962](https://tools.ietf.org/html/rfc4962) and
  [RFC 6819](http://tools.ietf.org/html/rfc6819)), such as fresh strong session
  keys, cryptographic algorithm independence, authentication of all parties
  involved (user, client app, server, etc), and proof against common attacks
  (replay attacks, man-in-the-middle, token hijacking and many others).
* Provided a familiar end-user experience
* Was usable on public or shared computers (had reasonable sign-out
  capabilities, etc)
* Was compatible with the other complementary Solid specs (such as the Web
  Access Control authorization system)
* Had support for down-the-road capability requirements, such as access
  revocation, blacklists and whitelists for both apps and service providers,
  the possibility of pairwise pseudonymity, pre-authorized "unattended" access,
  and so on.

### Why OpenID Connect

We have found OpenID Connect the only protocol that fit all of those
requirements (or even came close). At its heart, OIDC is a general purpose
toolkit for authentication and delegation, with provisions and specifications
for:

- User authentication via methods both familiar (such as username and password,
  WebID-TLS browser side certificates) and upcoming (such as the FIDO 2/
  [WebAuthn](https://www.w3.org/TR/webauthn/) standard)
- Verification of identity providers
- Verification of client applications of all types (browser side,
  server side, desktop and mobile, etc)
- Session expiration, user-initiated logout, and access revocation
