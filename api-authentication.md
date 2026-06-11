# Accessing the Digital Weight Management Programme Provider API with Microsoft Entra ID certificate authentication

This document describes the steps required to call the **Digital Weight Management Programme Provider API** 

The API uses JWT bearer authentication. Customer systems obtain bearer access tokens from Microsoft Entra ID by using the OAuth 2.0 client credentials flow with a client certificate. The customer creates and owns the key pair, shares only the public certificate with the API team, and keeps the private key secure.

## API contract summary

The API OpenAPI specification declares:

| Item | Value |
| --- | --- |
| Security scheme | HTTP bearer |
| Bearer format | JWT |
| Required API version header | `X-Api-Version: 3` |
| Request content type for write operations | `application/json` |

Every API request must include:

```http
Authorization: Bearer <access_token>
X-Api-Version: 3
Accept: application/json
```

Write operations with request bodies must also include:

```http
Content-Type: application/json
```

## Authentication model

The integration uses the OAuth 2.0 client credentials flow. The customer system acts as a confidential client and requests an app-only access token from Microsoft Entra ID. The customer proves its identity by signing a JWT client assertion with the private key that corresponds to the public certificate registered on the Microsoft Entra app registration.

The API receives the access token in the HTTP `Authorization` header:

```http
Authorization: Bearer <access_token>
```

The Provider API validates the token before serving the request.

## Connection overview

To connect to the Provider API, the customer must:

1. Generate and securely store their own private/public key pair.
2. Send only the public certificate to the API team for registration.
3. Keep the private key secret and never send it to the API team.
4. Receive the required Microsoft Entra ID and Provider API connection values from the API team.
5. Use the private key to authenticate the client application to Microsoft Entra ID.
6. Acquire access tokens using the client credentials flow.
7. Send the returned access token as a bearer token when calling the Provider API.
8. Include `X-Api-Version: 3` on Provider API calls.
9. Monitor certificate expiry and coordinate certificate rollover before the active certificate expires.

## Values supplied by the API team

Before attempting to access the API, confirm that you have received these values:

| Value | Description |
| --- | --- |
| Tenant ID | Microsoft Entra tenant ID used to issue access tokens. |
| Client application ID | Application/client ID assigned to your customer client application. |
| Token endpoint | `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`. |
| Provider API Application ID URI | Resource identifier for the Digital Weight Management Programme Provider API. |
| Scope | `<provider-api-application-id-uri>/.default`. |
| Provider API base URL | Base URL for the Provider API App Service. |

## Customer certificate requirements

The customer should create an X.509 certificate with a key size of 4096 suitable for client authentication.

The customer must share the public certificate file only, such as a `.cer`, `.pem`. The private key must remain under the customer's control.

Where possible, the customer should use a Microsoft Authentication Library (MSAL) for their platform instead of manually constructing the JWT assertion.

## Customer implementation steps

### 1. Store configuration

The customer application needs these values:

| Setting | Description |
| --- | --- |
| `TenantId` | Microsoft Entra tenant ID provided by the API team. |
| `ClientId` | Application/client ID for the customer client app registration. |
| `CertificatePrivateKey` | Customer-held private key or certificate reference. |
| `CertificateThumbprint` | Thumbprint for the public certificate uploaded to the app registration. |
| `TokenEndpoint` | `https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token`. |
| `Scope` | `<provider-api-application-id-uri>/.default`. |
| `ProviderApiBaseUrl` | Base URL for the Digital Weight Management Programme Provider API App Service. |
| `ApiVersionHeader` | `X-Api-Version: 3`. |

For client credentials flows, Microsoft advises using a tenant-specific authority rather than `common` or `organizations`.

### 2. Acquire an access token

Preferred approach: use MSAL and a certificate credential.

Conceptual .NET example:

```csharp
using Microsoft.Identity.Client;
using System.Security.Cryptography.X509Certificates;

var tenantId = "<tenant-id>";
var clientId = "<customer-client-application-id>";
var scope = "<provider-api-application-id-uri>/.default";
var certificate = new X509Certificate2("<path-to-certificate-with-private-key>", "<certificate-password>");

var app = ConfidentialClientApplicationBuilder
    .Create(clientId)
    .WithAuthority($"https://login.microsoftonline.com/{tenantId}")
    .WithCertificate(certificate)
    .Build();

var result = await app
    .AcquireTokenForClient(new[] { scope })
    .ExecuteAsync();

var accessToken = result.AccessToken;
```

Protocol-level request shape:

```http
POST https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id=<customer-client-application-id>
&scope=<provider-api-application-id-uri>/.default
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=<signed-client-assertion-jwt>
&grant_type=client_credentials
```

Microsoft Entra ID returns a bearer access token when the certificate assertion is valid and the client app is authorised for the Provider API.

### 3. Call the Provider API

Example: get a referral status.

```http
GET https://<provider-api-app-service-host>/referral/GP0000000001/status
Authorization: Bearer <access_token>
X-Api-Version: 3
Accept: application/json
```
Do not parse or depend on access token contents in the customer client application. The provider API receiving the token is responsible for validating the token and inspecting claims.

### 4. Cache and renew tokens

Access tokens expire. The customer application should cache tokens and request a new token when needed. MSAL handles token caching for `AcquireTokenForClient`; do not request a new token for every Provider API call.

## Certificate rollover

To avoid outages:

1. Customer generates a new key pair before the active certificate expires.
2. Customer sends the new public certificate to the API team for registration.
3. Customer waits for confirmation that the new certificate is registered and available for token requests.
4. Customer deploys the application change to use the new private key/certificate.
5. Customer validates that token acquisition and Provider API calls succeed with the new certificate.
6. Customer agrees a date with the API team for removal of the old certificate after successful cutover.

## Troubleshooting checklist

| Symptom | Checks |
| --- | --- |
| Token request fails | Confirm tenant ID, client ID, certificate thumbprint, assertion audience, assertion expiry, and that the public certificate is uploaded to the app registration. |
| Invalid scope | Confirm the scope is exactly `<provider-api-application-id-uri>/.default` and refers to the Provider API resource. |
| Provider API returns 401 | Confirm the bearer token is present in `Authorization`, the token is not expired, the token was issued by the expected tenant, and App Service or API middleware trusts the issuer and audience. |
| Provider API returns 403 | Confirm with the API team that the customer client application has the required Provider API app role/application permission and that admin consent or app role assignment is complete. |
| Provider API returns 404 | Confirm `providerUbrn` is 12 characters, the referral exists, and any `{date}` path value uses `yyyy-MM-dd`. |
| Provider API returns 422 | Confirm the submitted date and request body satisfy the business rules described in the OpenApi specification. |
| Intermittent failures near expiry | Confirm token caching, clock synchronisation, certificate validity dates, and rollover status. |

## Microsoft documentation references

- [Microsoft identity platform and the OAuth 2.0 client credentials flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)
- [Microsoft identity platform application authentication certificate credentials](https://learn.microsoft.com/en-us/entra/identity-platform/certificate-credentials)
- [Client credential flows - Microsoft Authentication Library for .NET](https://learn.microsoft.com/en-us/entra/msal/dotnet/acquiring-tokens/web-apps-apis/client-credential-flows)
- [How to configure daemon apps that call web APIs](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-daemon-app-configuration)
- [Protected web API: Code configuration](https://learn.microsoft.com/en-us/entra/identity-platform/scenario-protected-web-api-app-configuration)
- [Create a self-signed public certificate to authenticate your application](https://learn.microsoft.com/en-us/entra/identity-platform/howto-create-self-signed-certificate)
