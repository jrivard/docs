# OAuth, SAML and Session/Logout Confusion

Jason Rivard https://github.com/jrivard/docs

License: [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode.en)

This document attempts to explain a confusing scenario regarding federated authentications on the web, multiple applications and logouts.  

## Scenario Description
* A user logs into multiple applications sharing a common web-federation IDP
* User logs out 
* Second user using same browser and has access to first user's applications

This scenario is not a bug in any app or service, it is fundamental to the way web-federated authentication such as OAuth and SAML works.

Our example scenario will have three services:

- IDP1  (OAuth or SAML "Identity Provider" or "Login Web Page")
- APP-A ( Application A )
- APP-B ( Application B )

And these users:

- Bob
- Alice

## The logout problem scenario

1. Using his web browser, Bob authenticates to APP-A and APP-B both via IDP1.  After authenticating, Bob  will have an established session (typically via browser cookie) with all three services.
2. Bob continues to use APP-A, and after a few minutes clicks the logout button.  APP-A's session is de-authenticated.  Now the browser has active sessions with IDP1 and APP-B.  In Bob's head, he has "logged-out" of all the things.  
3. If the app is well-behaved, the app will redirect the user's browser to IDP1 so it can also process the logout request.  The IDP will de-authenticate the user's session.  However, the browser still has an active session with Bob authenticated to APP-B.
4. Bob walks away from his browser and lets Alice use his computer - in his head, he has "logged-out" so has no reason for concern.
5. Alice clicks on APP-A, logs in to the IDP and APP-A.  
6. Alice clicks on APP-B and she discovers that she is already logged in, but logged in as Bob to APP-B.  
7. Panic!

Note that this scenario has a couple scary points above, but it does require physical access to the same machine (and browser.)  

Some thoughts about the above sequence:

* This scenario requires shared physical access to the same machine and browser.
* Two applications are described, but typically there are many more, and so Alice could have access to all the other apps that Bob previously authenticated to.
* The time frames involved could be minutes to years, the controlling function is the maximum duration of an application's session timeout.
* In step 3 above, if the app is not well-behaved, the browsers next request to APP-A would just log the user in as Bob again re-using Bob's existing IDP session.
* This scenario is most applicable when the users think of multiple web apps as a single app.  In enterprise environments or co-branding scenarios this is a common situation. 
* A fundamental property of this issue is that the IDP does not have knowledge of application session existence or status. 
* This description focuses on web-based federation, but non-web or partial web-models are also subject to similar issues.

## What about single-logout?

SAML and OAuth both have protocols to process logout requests (SLO), but in practice they are not commonly used, and are not 100% reliable.  Since the authentication process requires the user's browser to participate in some type of token exchange, so would any logout process.  Thus, the user's browser is also required to participate in de-authentication.  It is common for apps to not even re-direct the browser to the IDP on an application logout request (step 2 above), but even if the browser does go to the IDP to logout, the IDP now has to get the browser to terminate the session with every app it has authenticated to.   Since browsers sandbox websites and generally try to prohibit the web app from initiating cross-domain requests, this is increasingly difficult and past tricks like hidden iframes/pixels are increasing less reliable. 

**SAML** offers a back-channel authentication process where the app and IDP exchange tokens directly, and this also allows for de-authentication with the browser and SLO can work well in this case (assuming it is well implemented in both app and IDP).  SAML back-channel single sign on is arguably the most secure of available federated login processes, but it is difficult to configure and rarely used.

**OAuth** has had a number of attempts to make a single-logout process standard over the years - I won't describe them here, but generally they are not that common.

The key challenges for single-logout to work are:
1) The logout problem is often ignored or not important for business reasons, so only a small segment of federation users care about it
2) Web browser feature and security model changes break the fragile logout processes

## Mitigations

### Ignore it

* Testers and admins will have to be aware of this scenario and properly terminate and not-share browser sessions between the multiple accounts they have access to, but normal users that only have a single account and do not share devices will never see this. 
* Do not allow/promote kiosk usage or other shared device models in your federated environment.  

## Implement Single-Logout

* All your apps and federation services need to support it
* Tends to be fragile and complex to configure
* Is unlikely to be 100% reliable
* Tends to require session-state synchroniation acrross any type of load-balancer or clustered services in your tech stack which can be complex and expensive

### Limit exposure

* Limit application session duration
  * If applications, particularly those with sensitive security functions, have a short session duration, then the time window between Bob logging out, and Alice getting access to Bob's authenticated sensitive app session is also limited.  
  * Use the shortest practical expiration time (perhaps only a few minutes) for security sensitive applications.
* Use short federation token expiration timeouts.
  * Well-behaved apps are supposed to honor the expiration time given by IDP during authentication.  This requires the application to re-check the IDP for an active user session more frequently and limits the time-window for exposure.
    * Many apps ignore the IDP's expiration values
    * This will add load to the IDP service
    * Depending on the protocol and auth method, it may interrupt the user's application context and be incompatible with natural use of the application.

### Use a proxy/federation service

* If all access to all apps is connected through a proxy/gateway/agent that is federation aware and can interrupt or disconnect application sessions when an IDP de-authentication occurs.
* Can be used to cover some applications while others use "pure" federation-based authentication.
* The system has to explicitly handle disconnecting the application's user sessions, typically by deleting or mangling the app's browser session cookie.
* Enterprise-level web-SSO products tend to have this functionality - though often with many caveats.
