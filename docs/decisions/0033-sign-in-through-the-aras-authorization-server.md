# 0033. Sign-in only through the Aras authorization server

- Status: Accepted
- Scope: MVP

## Context

Every call to Aras is made as the signed-in user ([ADR 0005](0005-user-level-access-through-aras-server.md)), so the product needs that user's Aras credential. It can get one by redirecting the browser to the installation's own authorization server, or by showing its own login-name and password form. A form makes the service see the password and makes the product responsible for authentication rules that Aras already enforces: single sign-on, multi-factor authentication, lockout. The product also needs one clear answer to which component receives the Aras token.

## Decision

The Aras installation's authorization server is the only sign-in path. The browser is redirected to Aras, which returns an authorization result to the service. The service's sign-in handler exchanges it for an access token (and a refresh token where the installation issues one) and passes it to the Tool Gateway. The browser holds only an opaque session cookie and never receives an Aras token. There is no login-name and password form, no fallback and no setting that selects a sign-in method. Aras Insight never sees a user's password.

An installation without an authorization server is not supported in the MVP. This is a deployment prerequisite (a sign-in client registered once by the Aras administrator) and a compatibility check.

## Alternatives considered

| Alternative | Why rejected |
| --- | --- |
| Authorization server preferred, password form as a fallback | The service would see the password, so the claim that it never does would fail; a second path to build and review. |
| Detect the sign-in method at run time | Same exposure, and the path in use depends on detection. |
| Browser interface completes the token exchange | An Aras token would reach code in the browser; only the service holds Aras credentials. |

## Consequences

- The installation's own single sign-on, multi-factor and lockout rules apply unchanged.
- Installations with no authorization server cannot use the product in the MVP.
- Deployment needs one client registration and a verified flow.

## Related

[Security and permissions](../architecture/security-and-permissions.md#identity-flow), [Chat App](../specs/chat-app.md), [Deployment and operations](../architecture/deployment-and-operations.md#aras-version-compatibility-to-verify), [ADR 0034](0034-credentials-encrypted-at-rest.md), [ADR 0035](0035-job-credential-lifecycle.md).
