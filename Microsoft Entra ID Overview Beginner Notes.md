# Microsoft Entra ID Overview: Beginner Notes

Oct 6, 2026 · @Salike Hassan

## 1. The big picture

Microsoft Entra ID is the cloud identity system that cloud apps trust to answer two questions: **who is this?** and **what are they allowed to do?** It was called Azure Active Directory (Azure AD) until 2023, and Azure AD Connect is now Microsoft Entra Connect.&#32;

Entra ID is **not** Active Directory running in the cloud. It has no domain controllers and no domains. Kerberos, NTLM and LDAP assume a network you control, and the internet is not one, so **Entra ID speaks web protocols instead.**

&#91;embedded content: before vs now · 2 flows\]

Before, a domain controller answered inside the office network. Now, cloud apps ask Entra ID over the internet using web protocols.

|                | On-prem Active Directory       | Microsoft Entra ID                             |
| -------------- | ------------------------------ | ---------------------------------------------- |
| Where it lives | Inside your office network     | In the cloud, reached over the internet        |
| Structure      | Domains and domain controllers | Tenants; no domain controllers                 |
| Protocols      | Kerberos, NTLM, LDAP           | OpenID Connect, OAuth 2.0, SAML, WS-Fed        |
| Typical use    | Windows PCs, file servers      | Microsoft 365, Azure, SaaS apps, your own APIs |

| Protocol | Plain-English job | Used where |
| --- | --- | --- |
| OpenID Connect (OIDC) | Who are you? (sign-in) | Modern web and mobile apps |
| OAuth 2.0 | What may this app do? (authorization) | APIs, app-to-app calls; your Function App setup |
| SAML / WS-Fed | Older enterprise single sign-on | Many SaaS apps |

Entra ID has about 3,500 SaaS apps pre-integrated in its gallery, so one corporate login works across all of them. That is why John calls it a federation broker.

## 2. Authentication vs authorization

Authentication (AuthN) proves who you are; authorization (AuthZ) decides what you may do. In one line: **AuthN checks the ID, AuthZ checks the permission.** John's key point is that authorization always happens in Entra ID, even if the password was checked somewhere else.

|  | Authentication (AuthN) | Authorization (AuthZ) |
| --- | --- | --- |
| Question | Who are you? | What are you allowed to do? |
| Applies to | A user, an app, or a device | That same identity, per resource |
| Analogy | Showing your ID at the airport gate | The boarding pass: which flight, which seat |
| Protocol | OpenID Connect | OAuth 2.0 |
| Proof | Password, MFA, certificate, client secret | Access token with scopes or roles |
| Where decided | Entra ID (or on-prem with pass-through or federation) | Always Entra ID |

Your Function App setup used both: the calling app proved itself with its client secret **(AuthN)**, and** the access token it received listed what it may call (AuthZ).**

## 3. Tokens and the sign-in flow

Entra ID hands out tokens instead of passwords. An **access token** is a short-lived pass for one specific service, and a **refresh token** quietly fetches new access tokens so you do not sign in again.

| Token | What it is | Lifetime | Who reads it |
| --- | --- | --- | --- |
| ID token | Proves who just signed in (OpenID Connect) | Short | The app that asked for sign-in |
| Access token | Pass for ONE API; says what the caller may do | About an hour | The API (for example your Function App) |
| Refresh token | Gets new access tokens without a new sign-in | Up to 90 days, renewed on use | Only Entra ID |

&#91;embedded content: sign-in and token flow · 8 steps\]

Steps 1 to 6 are the first call. Steps 7 and 8 show why the refresh token exists: the client gets a pass for a second service without signing in again.

**Apps are different from people.** When an app calls an API with no human involved, it uses the *client credentials flow*: it sends its client ID and a secret (or certificate) and receives an access token. There is **no refresh token** in this flow; the app simply asks again when the token expires. This is what the caller of your secured Function App does.

## 4. Tenant vs subscription

**A ****tenant**** is your organization's identity directory.** A **subscription** is a billing and resource container that trusts exactly one tenant. One tenant can hold many subscriptions, and a subscription can be moved to a different tenant, but it trusts only one at a time.

&#91;embedded content: tenant above subscriptions · trust arrows\]

The arrows point from each subscription to the tenant it trusts, because identities live in the tenant and resources live in the subscription.

Entra ID and Azure resources have **separate permission systems**, which is a common source of confusion:

|  | Entra roles | Azure RBAC roles |
| --- | --- | --- |
| Controls | Identity: users, groups, app registrations, directory settings | Azure resources: Function Apps, Redis, VMs |
| Example roles | Global Administrator, Application Administrator | Owner, Contributor, Reader |
| Scope | The whole tenant | Management group, subscription, resource group or single resource |

## 5. Conditional Access and MFA

Conditional Access is a set of if-then rules that Entra ID runs every time someone asks for a token. It is how MFA gets enforced, and it is where "authorization always happens in Entra ID" becomes visible.

&#91;embedded content: Conditional Access decision flow\]

A request that matches no policy is allowed. A match applies the controls, and the token is issued only once they are met.

John's guidance on MFA: do not prompt on every sign-in. Constant prompts train users to tap "yes" without thinking, which helps an attacker. Prompt when something looks risky, such as a new location, an unusual sign-in, or a privileged role being activated. Windows Hello for Business counts as strong authentication, so it satisfies MFA on its own.

| Licence | What you get |
| --- | --- |
| Free | Basic sign-in and security defaults |
| P1 | Conditional Access and MFA policies; the enterprise minimum |
| P2 | Identity Protection (risk detection), Privileged Identity Management, access reviews |

**Test safely:** when trying a new policy, exclude an emergency admin account. A wrong policy aimed at all users and all apps can lock you out of your own tenant.

## .6 On-prem to cloud sync (background only)

Companies that already run Active Directory copy their users into Entra ID with **Microsoft Entra Connect**. On-prem AD stays the source of truth; Entra ID gets a matching cloud object for each user. You do not need to configure this now, but enterprise clients will mention it.

The remaining choice is where the password is checked at sign-in:

| Option | Where the password is checked | John's view |
| --- | --- | --- |
| Cloud authentication (password hash sync) | In Entra ID, using a hash of the hash | Recommended |
| Pass-through authentication | On-prem domain controllers, through agents | Middle option |
| Federation (AD FS) | On-prem federation servers | Old way, heavy to maintain |

Cloud authentication wins because nothing on-prem has to stay running, and Microsoft can match leaked passwords found on the dark web against the stored hash. Once you have a refresh token, none of the three options is contacted again until it expires, so federation adds maintenance without much benefit.

## 7. Guests: B2B vs B2C

B2B is for people you work **with**, and B2C is for customers who use **your** app.

|  | B2B (collaboration) | B2C (customer identity) |
| --- | --- | --- |
| Who | Partners, vendors, contractors | Your app's customers |
| How they sign in | Their own account: another Entra tenant, Microsoft account, Gmail, or a one-time email code | Social logins or local accounts you offer |
| Where they live | As guest users in your tenant | In a separate B2C directory |
| Who manages the password | Their home provider, not you | The B2C service |
| Authorization and MFA | Your tenant decides; their MFA is not trusted automatically | Its own MFA, billed per authentication |

B2B licensing is generous: each P1 licence in your tenant covers up to five guest users. For your consulting work, B2B guest access is one way to let your engineers work in a client tenant. The other is a service principal and service connection, which is topic 5 on your list.

## 8. What matters for your six learning topics

Sections 2, 3 and 4 are the foundation for everything on your list; the rest is background until a client or exam needs it.

| Concept from this overview | Priority | Feeds into |
| --- | --- | --- |
| AuthN vs AuthZ, tokens, client credentials flow | Must know now | App-to-app authentication and authorization |
| Tenant vs subscription, Entra roles vs Azure RBAC | Must know now | Both service connection topics, especially cross-tenant |
| App registration vs service principal vs enterprise app | Must know next | Service principal, service connections |
| Conditional Access and MFA | Needed later | MFA topic |
| B2B guest users | Useful | Cross-tenant work |
| Entra Connect, pass-through, federation | Background | Client conversations |
| B2C, PIM, access reviews | Later | Governance and security work |

## 9. Check questions

Cover the answers and try each question in your own words first. Your original answers were strong; the notes under each show what to sharpen.

**Q1. What is the difference between authentication and authorization?**

- Model answer: authentication proves who you are (user, app or device); authorization decides what that identity may do. AuthN checks the ID, AuthZ checks the permission.
- Your answer: right idea. Sharpen "authentication = who" to "authentication = *prove* who".

**Q2. Why does Entra ID not speak Kerberos?**

- Model answer: Kerberos and NTLM assume a domain and a network you control. The cloud has neither, so Entra ID uses internet-friendly protocols: OpenID Connect, OAuth 2.0 and SAML.
- Your answer: correct. You said it is not domain specific and not part of Active Directory, which is exactly the point.

**Q3. What is the difference between an access token and a refresh token?**

- Model answer: an access token is a short-lived pass for one service and is what APIs check; a refresh token is long-lived and only gets new access tokens without a fresh sign-in.
- Your answer: right. Two fixes: the access token is mainly about *authorization*, because proving who signed in is the ID token's job in OpenID Connect. And the client credentials flow used by apps has no refresh token at all.

**Q4. Is a tenant inside a subscription, or the other way around?**

- Model answer: the tenant is above. One tenant can have many subscriptions, and each subscription trusts exactly one tenant.
- Your answer: correct. Add the last half: each subscription trusts only one tenant at a time.

## 10. References and next lesson

I opened each of these pages to check it. The Microsoft overview page has moved since my earlier message, so use the link below, not the older `/whatis` address.

| Resource | What it covers | Use it for |
| --- | --- | --- |
| [What is Microsoft Entra?](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra) | The Entra product family, including Entra ID and Workload ID (identities for apps and pipelines) | The big picture; where pipeline identities fit |
| [Application and service principal objects in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals) | App registration vs service principal, with a worked multi-tenant example | The next lesson |
| [Identity and Access Administrator (SC-300)](https://learn.microsoft.com/en-us/credentials/certifications/identity-and-access-administrator/) | Exam page with study guide and practice assessment; the exam includes workload identities | A structured path, even if you never sit the exam |
