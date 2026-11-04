# Pattern: the issuer and an existing identity provider

Most Companies already sign customers in through an identity provider, often a customer identity
(CIAM) service. Poppy sign-in is plain OAuth
([4](https://personalagentprotocol.org/docs/0.3/spec#oauth)), so that provider can usually stay in place.
The question is which server Personal Agents treat as the Company's issuer
([3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server)):

- **The identity provider is the issuer.** Personal Agents get their tokens from the provider your
  website and apps already use.
- **An issuer for Personal Agents, alongside the identity provider.** A separate OAuth server
  handles Personal Agents, and sends Users to your identity provider to sign in. Your identity
  provider stays the source of accounts and sign-in methods.

Both keep the User's sign-in, MFA, and fraud checks where they are today. Signed-out Sessions need
neither: they rely on the Session header alone, and no identity provider sees them
([5.3](https://personalagentprotocol.org/docs/0.3/spec#session-verification)).

```mermaid
flowchart TD
  start([Do tasks need the User's account?]) -->|No| none[No issuer. Signed-out Sessions only.<br/>Identifier is https://domain]
  start -->|Yes| gaps{Does the identity provider support<br/>everything in spec 4.8?}
  gaps -->|Yes, or with configuration| idp[The identity provider is the issuer]
  gaps -->|No| front{Is the only gap Mediated Sign-In<br/>or extra metadata fields?}
  front -->|Yes| shim[The identity provider is the issuer,<br/>with a small handler in front]
  front -->|No| agent[An issuer for Personal Agents,<br/>federated to the identity provider]
  idp --> ready{Ready to let Personal Agents sign in?}
  shim --> ready
  agent --> ready
  ready -->|Not yet| disabled[Keep the issuer, set auth.disabled]
  ready -->|Yes| live[Offer sign-in]
```

**Choose before launch.** User IDs, Sessions, and refresh tokens belong to the issuer
([3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server)). Moving from one pattern to the
other later changes the Company's identifier, so every User signs in again and earlier Sessions
don't carry over. This applies even to a Company that offers only signed-out Sessions today, if
its domain's protected resource metadata already names the identity provider: that provider is
already its identifier.

## What the issuer has to support

Check the identity provider against this list in the survey. Each row is something
[4.8](https://personalagentprotocol.org/docs/0.3/spec#oauth-additions) adds to OAuth, or a requirement
of [3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server).

| Requirement                                                                        | Spec                                                                                   | If the identity provider can't                                                                           |
| ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Clients identified by a client ID metadata document URL                            | [4.1](https://personalagentprotocol.org/docs/0.3/spec#agent-identity)                      | Register each Personal Agent you accept by its `client_id` URL, and reject others with `invalid_client`. |
| `private_key_jwt` with keys from `jwks_uri`, `ES256` required                      | [4.1](https://personalagentprotocol.org/docs/0.3/spec#agent-identity)                      | An issuer for Personal Agents.                                                                           |
| Client assertion `aud` exactly the issuer, rejecting the token endpoint URL        | [4.1](https://personalagentprotocol.org/docs/0.3/spec#agent-identity)                      | An issuer for Personal Agents.                                                                           |
| `iss` in every authorization response, including errors (RFC 9207)                 | [4.2](https://personalagentprotocol.org/docs/0.3/spec#direct-sign-in)                      | An issuer for Personal Agents.                                                                           |
| PKCE with `S256`, exact `redirect_uri` matching, loopback ports (RFC 8252)         | [4.2](https://personalagentprotocol.org/docs/0.3/spec#direct-sign-in)                      | An issuer for Personal Agents.                                                                           |
| A refresh token from every sign-in, rotation as RFC 6749 §6 allows                 | [4.4](https://personalagentprotocol.org/docs/0.3/spec#refresh-tokens)                      | Usually configuration. Otherwise an issuer for Personal Agents.                                          |
| No scopes beyond those requested, and at least one granted                         | [4.7](https://personalagentprotocol.org/docs/0.3/spec#scopes)                              | Usually configuration. Otherwise an issuer for Personal Agents.                                          |
| RFC 7009 revocation, listed as `revocation_endpoint`                               | [4.6](https://personalagentprotocol.org/docs/0.3/spec#revocation)                          | An issuer for Personal Agents.                                                                           |
| `grant_types_supported` with `refresh_token`, and `protected_resources` (RFC 9728) | [3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server)                        | If the issuer is on your own domain, a handler can serve the metadata with these fields. See below.      |
| Resources can tell which `client_id` a token was issued to                         | [5.3](https://personalagentprotocol.org/docs/0.3/spec#session-verification)                | A claim such as `client_id` or `azp`, or introspection.                                                  |
| Resource indicators (RFC 8707), for MCP servers                                    | [4.5](https://personalagentprotocol.org/docs/0.3/spec#access-tokens)                       | Needed only for services that accept only their own tokens.                                              |
| The device authorization grant, if you offer Device Sign-In                        | [4.3](https://personalagentprotocol.org/docs/0.3/spec#device-sign-in)                      | Offer Direct Sign-In only, or an issuer for Personal Agents.                                             |
| A custom grant type, if you offer Mediated Sign-In                                 | [Mediated Sign-In](https://personalagentprotocol.org/docs/0.3/extensions/mediated-sign-in) | A grant handler in front of the token endpoint. See below.                                               |
| DPoP-bound access tokens (optional)                                                | [4.5](https://personalagentprotocol.org/docs/0.3/spec#access-tokens)                       | Issue Bearer tokens. Verifying the Session header and its `ath` binds them to the Personal Agent's key.  |

Record each row's answer, with evidence, in `plan/survey.md`.

## The identity provider is the issuer

Personal Agents are one more kind of OAuth client at the provider your website already uses.

- Fits: an identity provider that supports the table above, or needs only configuration.
- Needs: the provider's metadata with the fields in
  [3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server), protected resource metadata on
  each domain naming the provider in `authorization_servers`, a consent page that shows the
  Personal Agent's `client_id` domain (`patterns/sign-in.md`), and Poppy's scopes in the provider.
- Watch for: an `openid` scope in `scopes_supported`. Personal Agents don't need ID tokens. Leave
  it out of the list, or describe it in `auth.scopes`
  ([4.7](https://personalagentprotocol.org/docs/0.3/spec#scopes)). Also watch for a provider that
  grants first-party apps' scopes by default. Limit what Personal Agents get by client, or with
  `unauthorized_client` ([guides](https://personalagentprotocol.org/docs/0.3/guides#company-agent-issuer)).

```mermaid
flowchart LR
  PA[Personal Agent] -->|poppy.json| Site[example.com]
  PA -->|protected resource metadata| Site
  PA -->|"authorize, token, revoke<br/>(client assertion aud = IdP issuer)"| IdP[Identity provider<br/>issuer: https://login.example.com]
  Apps[Website and first-party apps] --> IdP
  PA -->|"Session header (aud = IdP issuer)<br/>+ access token"| GW[Gateway]
  GW --> API[APIs, MCP, conversations]
  GW -.->|validate tokens| IdP
```

Direct Sign-In then looks like this. The User signs in with any method the provider already offers
([4.2](https://personalagentprotocol.org/docs/0.3/spec#direct-sign-in)).

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant B as User's browser
  participant IdP as Identity provider (issuer)
  participant CM as Personal Agent's client metadata
  PA->>B: Open authorization_endpoint (client_id URL, scope, state, PKCE)
  B->>IdP: GET /authorize
  IdP->>CM: Fetch client_id URL and jwks_uri, check redirect_uri
  IdP->>B: Existing sign-in (password, passkey, MFA)
  IdP->>B: Consent page (agent name, client_id domain, scopes)
  B->>IdP: Approve some or all scopes
  IdP->>B: Redirect to redirect_uri with code, state, iss
  B->>PA: code, state, iss
  PA->>PA: Check iss is the issuer it started with
  PA->>IdP: Token request (code, code_verifier, client assertion aud = issuer)
  IdP->>PA: access_token, refresh_token, scope
```

### Small handlers in front of the identity provider

Two gaps can be closed without a second issuer:

- **Metadata fields.** If the issuer is on a domain you control, such as a custom domain for the
  provider, a handler at the RFC 8414 metadata URL can serve the provider's metadata with
  `protected_resources` and `grant_types_supported` added. The `issuer` and every endpoint stay
  exactly as the provider serves them. Keep it in step with the provider's own metadata.
- **Mediated Sign-In.** A grant handler at the token endpoint recognizes the Mediated Sign-In grant
  type, checks credentials through the provider's API, and returns tokens the provider issues for
  that account. It works only if the provider can issue tokens for a sign-in your server completed.
  It passes every other request to the provider unchanged (`patterns/sign-in.md`).

A gap in the authorization flow itself, such as `iss` in redirects or the client assertion's
audience, needs an issuer for Personal Agents instead. A proxy that rewrites authorization requests
and responses is an issuer in all but name, and harder to keep correct.

## An issuer for Personal Agents, alongside the identity provider

A separate OAuth server, such as `https://agents.example.com`, is the Company's issuer for Personal
Agents. It meets [4.8](https://personalagentprotocol.org/docs/0.3/spec#oauth-additions) itself, and it
signs Users in through your identity provider with OpenID Connect, as one more client of the
provider. Your website, apps, and accounts keep using the identity provider directly.

- Fits: an identity provider that can't meet the table above, or a Company that wants Personal
  Agent clients, consent, and tokens kept apart from first-party ones.
- Needs: the new server's metadata and the domain's protected resource metadata
  ([3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server)); an OpenID Connect client at the
  identity provider for the new server; a stable account ID from the provider in the new server's
  tokens; resources that accept the new server's tokens; and revocation when the account changes
  (below).
- Watch for: the identifiers. The Company's identifier is the new issuer, so Session JWTs use it as
  their audience and User IDs are scoped by it
  ([5.1](https://personalagentprotocol.org/docs/0.3/spec#user-ids)). Map the provider's stable user ID
  to your account, never an email address, which can change or be reused.

```mermaid
flowchart LR
  PA[Personal Agent] -->|poppy.json, protected resource metadata| Site[example.com]
  PA -->|"authorize, token, revoke<br/>(client assertion aud = agent issuer)"| AS[Issuer for Personal Agents<br/>https://agents.example.com]
  AS -->|OpenID Connect sign-in| IdP[Identity provider]
  Apps[Website and first-party apps] --> IdP
  PA -->|"Session header (aud = agent issuer)<br/>+ access token"| GW[Gateway]
  GW --> API[APIs, MCP, conversations]
  GW -.->|validate agent tokens| AS
  Apps -->|identity provider tokens| GW
  GW -.->|validate first-party tokens| IdP
```

### Discovery with two authorization servers

If the domain's protected resource metadata lists only the new issuer, Personal Agents use it. If it
lists both servers, for example because other clients find the identity provider there, `auth.issuer`
in `poppy.json` names the one Personal Agents use
([3.1](https://personalagentprotocol.org/docs/0.3/spec#fields)). Either way, Personal Agents check
`protected_resources` only in the new issuer's metadata, so it has to list the domain.

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant Site as example.com
  participant AS as Issuer for Personal Agents
  PA->>Site: GET /.well-known/poppy.json
  Site->>PA: organization.domain = example.com, auth.issuer = https://agents.example.com
  PA->>Site: GET /.well-known/oauth-protected-resource
  Site->>PA: authorization_servers = [https://login.example.com, https://agents.example.com]
  PA->>PA: Use auth.issuer, which is in the list
  PA->>AS: GET /.well-known/oauth-authorization-server
  AS->>PA: issuer, endpoints, grant_types_supported, protected_resources
  PA->>PA: Check issuer matches exactly, and protected_resources has https://example.com
```

### Direct Sign-In through the identity provider

The new issuer's authorization endpoint sends the User to the identity provider to sign in, then
shows its own consent page. The identity provider's sign-in page can be on any host
([4.2](https://personalagentprotocol.org/docs/0.3/spec#direct-sign-in)).

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant B as User's browser
  participant AS as Issuer for Personal Agents
  participant IdP as Identity provider
  PA->>B: Open agent issuer's authorization_endpoint (client_id URL, scope, state, PKCE)
  B->>AS: GET /authorize
  AS->>AS: Fetch and check client metadata, keep the request
  AS->>B: Redirect to identity provider (its own OIDC client, state, PKCE)
  B->>IdP: Existing sign-in (password, passkey, social, MFA)
  IdP->>B: Redirect to agent issuer's callback with code
  B->>AS: code
  AS->>IdP: Exchange code for ID token
  IdP->>AS: ID token with sub (the account)
  AS->>B: Consent page (agent name, client_id domain, account, scopes)
  B->>AS: Approve some or all scopes
  AS->>B: Redirect to redirect_uri with code, state, iss = agent issuer
  B->>PA: code, state, iss
  PA->>AS: Token request (code, code_verifier, client assertion aud = agent issuer)
  AS->>PA: access_token, refresh_token, scope (tokens carry the account and client_id)
```

The identity provider may skip its sign-in step when the browser is already signed in. The consent
page still appears on every Poppy sign-in, so the User approves the Personal Agent's scopes each
time.

### Device Sign-In through the identity provider

The verification page has to be on the new issuer's host, or on the host of a URL in its
`protected_resources`, or a subdomain of either. Personal Agents check this before showing the link
([4.3](https://personalagentprotocol.org/docs/0.3/spec#device-sign-in)). The page can then send the
User to the identity provider.

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant U as User's phone
  participant AS as Issuer for Personal Agents
  participant IdP as Identity provider
  PA->>AS: POST device_authorization_endpoint (scope, client assertion)
  AS->>PA: device_code, user_code, verification_uri on agents.example.com
  PA->>PA: Check the link's host, show it with organization.name and domain
  PA->>U: Link or QR code, and user_code
  U->>AS: Open verification_uri (doesn't approve)
  AS->>U: Redirect to identity provider
  U->>IdP: Existing sign-in
  IdP->>AS: ID token with sub
  AS->>U: Confirm the code matches, consent page
  U->>AS: Approve
  loop Every interval seconds
    PA->>AS: Token request (device_code, client assertion)
  end
  AS->>PA: access_token, refresh_token, scope
```

### Mediated Sign-In through the identity provider

If you offer the extension, the new issuer's grant handler checks credentials and sends one-time
codes through the identity provider's APIs
([Mediated Sign-In](https://personalagentprotocol.org/docs/0.3/extensions/mediated-sign-in)). Never
store the credentials or codes, including in logs.

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant AS as Issuer for Personal Agents
  participant IdP as Identity provider
  PA->>AS: Token request (mediated-sign-in grant, scope, fields, client assertion)
  AS->>IdP: Check credentials
  IdP->>AS: Valid, needs a one-time code
  AS->>IdP: Send a one-time code to the User
  AS->>PA: 400 code_required, sign_in_id, sent_to
  PA->>AS: Token request (sign_in_id, one_time_code, client assertion)
  AS->>IdP: Check the code
  IdP->>AS: Valid, account sub
  AS->>PA: access_token, refresh_token, scope
```

### Resources accept both issuers' tokens

Resources that Personal Agents call accept the new issuer's tokens, in addition to the identity
provider's tokens from your own apps. Verify both at a gateway, then pass the verified account and
scopes to the services behind it
([guides](https://personalagentprotocol.org/docs/0.3/guides#company-internal)). If services behind it
accept only identity provider tokens, the gateway can exchange a verified agent token for an
internal credential that allows no more than the agent token's scopes.

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant GW as Gateway
  participant AS as Issuer for Personal Agents
  participant API as API
  PA->>GW: GET /orders with Poppy-Session (aud = agent issuer) and access token
  GW->>GW: Verify Session JWT: client metadata, delegation, signature, aud, htm, htu, iat, ath
  GW->>AS: Validate access token (JWKS or introspection)
  AS->>GW: Valid: account, scopes, client_id
  GW->>GW: Check the token's client_id equals the Session JWT's iss, and the scopes
  GW->>API: Verified client_id, sid, User ID, account, scopes
  API->>GW: 200
  GW->>PA: 200
```

### Browser join

Your website accepts the new issuer's access tokens at `{browser_session_endpoint}/authorize`
([section 6](https://personalagentprotocol.org/docs/0.3/spec#web-browsing)). It then sets its own
Session cookie and checks that cookie before the identity provider's login, so the Personal Agent's
browser acts with the token's account and scopes.

```mermaid
sequenceDiagram
  autonumber
  participant PA as Personal Agent
  participant BR as Personal Agent's browser
  participant W as Website
  participant AS as Issuer for Personal Agents
  BR->>W: GET browser_session_endpoint
  W->>BR: Form with code, __Host-poppy_join cookie
  PA->>W: POST /authorize (code) with Poppy-Session and agent access token
  W->>AS: Validate access token
  AS->>W: Valid: account, scopes, client_id
  W->>PA: 204
  BR->>W: POST form (code) with join cookie
  W->>BR: Session cookie, 303 to a page
  BR->>W: Browse as the Session, with the token's account and scopes
```

### Keeping the account in step

Users manage their account at the identity provider, and their Personal Agents at the new issuer.
Connect the two:

- List connected Personal Agents in your account settings from the new issuer's records, and
  revoke there when the User disconnects one
  ([guides](https://personalagentprotocol.org/docs/0.3/guides#company-account-settings)).
- Revoke the account's refresh tokens at the new issuer when the identity provider reports a
  password reset, a suspected compromise, or a deleted or locked account, for example from its
  event webhooks ([guides](https://personalagentprotocol.org/docs/0.3/guides#company-lifetimes)).

```mermaid
sequenceDiagram
  autonumber
  participant U as User
  participant W as Account settings
  participant IdP as Identity provider
  participant AS as Issuer for Personal Agents
  participant PA as Personal Agent
  U->>W: Disconnect a Personal Agent
  W->>AS: Revoke its refresh tokens for this account
  U->>IdP: Reset password
  IdP->>AS: Event: password reset for sub
  AS->>AS: Revoke every refresh token for the account
  PA->>AS: Refresh
  AS->>PA: invalid_grant
  PA->>PA: Continue signed out
```

## Several Companies on one identity provider

A provider that hosts several Companies gives each its own issuer, such as
`https://agents.provider.example/shop-a`, even on shared servers
([3.2](https://personalagentprotocol.org/docs/0.3/spec#oauth-server),
[guides](https://personalagentprotocol.org/docs/0.3/guides#company-hosting)). Each issuer's
`protected_resources` lists only its own Company's domains, and a token from one never works at
another.
