---
title: "Working with ServiceNow authorization"
slug: "servicenow-auth"
hidden: false
category: "6554e8e7ecc618083df24cd5"
parentDocSlug: "servicenow-integration-alt"
---
Before your OpenFin app can connect to your ServiceNow instance, it must first get authorization by using the [OAuth authorization code grant flow](https://docs.servicenow.com/bundle/vancouver-platform-security/page/administer/security/concept/c_OAuthAuthorizationCodeFlow.html), which implements the [PKCE OAuth 2.0 flow](https://datatracker.ietf.org/doc/html/rfc7636) (Proof Key for Code Exchange, pronounced "pixy").
This flow provides secure authorization without needing to expose a sensitive client secret to insecure clients.
OpenFin's ServiceNow integration API handles the authorization process and token exchange automatically.
To guard against potential [XSS](https://developer.mozilla.org/en-US/docs/Glossary/Cross-site_scripting) vulnerabilities, OpenFin's integration includes a web worker that securely handles authorization tokens and remote requests that involve tokens.

[block:callout]
{
  "type": "info",
  "title": "About web workers",
  "body": "[Web workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers) are essentially background threads &mdash; that is, they run in a context that is different from the parent app's thread. This means the code running in the parent app cannot access code running in the web worker and vice versa. The only way a web worker and the parent app communicate is by exchanging messages, and if the web worker never leaks tokens through messages, we can consider the tokens secure from XSS."
}
[/block]

## The authorization process

Whenever the `connect` function is called from the ServiceNow integration API, a child window is created and navigated to the authorization endpoint that starts the OAuth 2.0 authorization code flow. The user may be prompted to authenticate, and then will be prompted to allow the app to connect to their ServiceNow account:

![ServiceNow dialog box, stating "My OpenFin App would like to connect to your ServiceNow account with "allow" and "deny" buttons.](https://cdn.openfin.co/docs/public/images/654321-working-with-servicenow-authorization-google-docs-0.png "An example of the ServiceNow dialog box.")

**It is important that they click the "Allow" button at this stage.**
If they click "Deny" or close the window, the authorization process will not complete successfully and the API will raise an `AuthorizationError`.

After the user approves the app's authorization request, the window is redirected to a predefined URL where it receives an authorization code.
The ServiceNow integration API then closes the window and exchanges the code for an [access token](https://oauth.net/2/access-tokens/) (for making REST API requests) and a [refresh token](https://oauth.net/2/refresh-tokens/) (for renewing expired access tokens).

At this point, the authorization process has completed successfully and the `connect` function returns with a ServiceNowConnection object, with which requests can be made to ServiceNow REST APIs.

## Token lifetimes

Access and refresh token lifetimes are configured with the OAuth registration in ServiceNow (see "Get Client ID and Instance Name" section). We have recommended sensible defaults but token lifetimes can be adjusted according to business requirements. 

## Handle token expiration

If a request is made to the ServiceNow REST API and the current access token has expired, the ServiceNow integration API automatically requests a new access token using the refresh token (stored in memory by the web worker).
If this fails (for example, if the refresh token has expired), then an `AuthTokenExpiredError` is thrown and the `connect` function must be called again to start the authorization process in order to retrieve new access and refresh tokens.

Whenever `executeApiRequest` is called, always catch errors and check to see whether the error is an `AuthTokenExpiredError`.
That way you can handle reconnection in the most ideal way for your users and particular environment configuration:

```typescript
import { AuthTokenExpiredError, ServiceNowEntities } from '@openfin/servicenow';

try {
  const response = await serviceNowConnection.executeApiRequest<ServiceNowEntities.FSO.FinancialServicesCase[]>('/api/now/v2/table/sn_bom_financial_service');
  // Do something with the response...
} catch (err) {
  if (err instanceof AuthTokenExpiredError) {
    // Need to reconnect...
  } else {
    throw err;
  }
}

```