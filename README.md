# Schemeful Same-Site Explainer

## Authors
* bingler@chromium.org
* davidben@chromium.org
* kaustubhag@chromium.org

## Introduction
The [SameSite](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.8) cookie attribute is designed to defend against CSRF attacks but currently does not take the scheme of the site into account. This was [originally to assist sites during their transition to https](https://github.com/w3c/webappsec-fetch-metadata/issues/34#issuecomment-527338651), however it results in the secure and insecure versions of the same host being considered same-site. A network attacker could thus impersonate http://site.example and use it to bypass `SameSite` protections on https://site.example. Between this security flaw and [HTTPS usage markedly increasing](https://transparencyreport.google.com/https/overview), we believe it is time to change this definition.

The web ecosystem is already moving in this direction with the [HTML spec](https://html.spec.whatwg.org/multipage/origin.html#same-site) having been updated with the new definition of same site and the in-progress W3C PING [Privacy Threat Model](https://w3cping.github.io/privacy-threat-model/#terminology) utilizing it.

### Example Attacks and Mitigations
Below are two scenerios: the first one illustrates how the `SameSite` cookie attribute (specifically `Strict` in this example) can protect against CSRF, the second illustrates how "Schemeful Same-Site" enhances that protection.
#### A Cross-Domain CSRF Attack
An attacker sends you an email claiming you've won a sweepstakes. All you need to do if visit a website and click a button. You decide this feels legitimate and go visit the site https://mega-sweepstakes.example. The site has a big attractive button urging you to click it but, unbeknownst to you, the button doesn't claim your prize. Instead it links to your bank along with a query parameter telling your bank to transfer your money to the attacker's account: https://bank.example/transfer/custom?from=checking&to=evil-inc-slush-fund&amt=1000. Oh, and you recently logged into your bank account so your login cookie is still valid.

If your bank's cookies don't have the `SameSite=Strict` attribute then clicking that link will cause your browser to navigate to your bank's site, send along your login cookies, and cause you to be $1000 poorer.

If your bank's cookies **do** have the `SameSite=Strict` attribute then clicking that link will **not** cause your browser to send along your login cookies, because mega-sweepstakes.example isn't the same site as bank.example, and your checking account is safe.

#### A Same-Domain CSRF Attack
But what if the attacker is Man-in-the-middling your connection? If you decide to visit http://bank.example the attacker could modify every link on the page to point to https://bank.example/transfer/custom?from=checking&to=evil-inc-slush-fund&amt=1000. Even `SameSite=Strict` wouldn't protect you as bank.example and bank.example are same-site.

Enter "Schemeful Same-Site", by considering both the scheme and the registrable domain we can see that http://bank.example and https://bank.example are not the same and hence are not same-site. This means that the browser will not send `SameSite=Strict` cookies, you will therefore not be logged in, and the transfer attempt will fail.

## Proposal
Modify same-site's implementation in the user agent to consider origins with different schemes as cross-site. Thus https://site.example and http://site.example would now be considered cross-site.

[Incrementally Better Cookies](https://tools.ietf.org/html/draft-west-cookie-incrementalism-01) has been updated to reflect this proposal.

## Goal
To protect users from CSRF attacks on secure origins which are carried out by insecure origins with the same domain.

## Non-Goals
This proposal does not address [weak confidentiality](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.5) or [weak integrity](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.6) of cookies, specifically in the context of scheme isolation. This means that both secure and insecure origins will retain access to the same set of cookies. I.e., if a user visits http://site.example with http://site.example/image.jpg and the response sets a cookie then when that user visits https://site.example the request to https://site.example/image.jpg will still send that same cookie.

"[Scheme-Bound Cookies](https://github.com/mikewest/scheming-cookies)" is a proposal aiming to address this issue.

## Recommended Action
Affected sites are encouraged to fully migrate to HTTPS.

## Changed Behavior
“Cross-scheme” in this context means origins which are [schemelessly](https://html.spec.whatwg.org/multipage/origin.html#schemelessly-same-site) same-site but have differing schemes. E.g., http://site.example and https://site.example are considered cross-scheme.

* `SameSite=Lax` and `SameSite=Strict` cookies will no longer be sent in cross-scheme subresource requests, i.e. http-to-https and https-to-http (mixed content), regardless of HTTP request method.
* `SameSite=Lax` and `SameSite=Strict` cookies can no longer be set by a cross-scheme subresource.
* `SameSite=Lax` and `SameSite=Strict` cookies will no longer be sent in cross-scheme top-level POSTs.
* `SameSite=Strict` cookies will no longer be sent in cross-scheme top-level navigations.

## Examples of Changed Behavior
Below are two common use cases which will be changing.

### Top-level POST

Cross-scheme top-level POSTS will no longer send `SameSite=Lax` cookies.

For example, a user writes up a forum post on http://messageboard.example. When the user clicks submit a POST request is sent to https://messageboard.example/submit, however the forum login cookie, which is marked with `SameSite=Lax`, will not be sent and the user’s post will fail.
### Subresources

Requests to cross-scheme subresources will no longer send `SameSite=Lax` or `SameSite=Strict` cookies. Similarly, responses from cross-scheme subresources will no longer be able to set such cookies.

For example, if http://site.example embeds an image from https://site.example, `SameSite=Lax` and `SameSite=Strict` cookies will no longer be sent on the request to https://site.example, and any such cookies in the response will be ignored. 

## Questions
### How do Schemeful Same-Site and Scheme-Bound Cookies differ?

"Schemeful Same-Site" and "[Scheme-Bound Cookies](https://github.com/mikewest/scheming-cookies)" are both trying to move cookies closer to an origin-based security model; these two proposals complement one another and one is not a subset of the other.

#### Schemeful Same-Site
"Schemeful Same-Site", and the current version of same-site, is concerned with the context in which the cookie will be sent or received, hereby referred to as the "same-site context". The "same-site context" can be thought of as the answer to the question "Is the request URL the same site as the one I'm on?" More specifically "Is the request URL 'same-site' or 'cross-site' with the current site I'm on?" 
"The site I'm on" is generally the one shown in the browser's address bar (there are, of course, exceptions but they're not important to illustrate the point).

In the current world the "same-site context" is determined by checking the registrable domain: http://site1.example is *not* same-site with http://site2.example, but http://site1.example and https://site1.example would be same-site.
With "Schemeful Same-Site" we now consider the scheme along with the registrable domain: http://site1.example is cross-site, not same-site, with http://site2.example, http://site1.example is also cross-site with https://site1.example, but https://site1.example is same-site with https://site1.example. 

#### Scheme-Bound Cookies
"Scheme-Bound Cookies" is concerned with the scheme of the URL a cookie is being sent to or set by.
In the current world a response from https://site.example/resource.jpg could set a cookie: `Set-Cookie: mycookie=value`.
That cookie can then be sent to any URL with "site.example" as the host regardless of the scheme: http://site.example/otherresource.jpg, https://site.example/somethingelse.jpg, etc.

With "Scheme-Bound Cookies" the browser will no longer send the cookie to a different schemes than the one that set it.
With that in mind: a response from https://site.example/resource.jpg sets a cookie: `Set-Cookie: mycookie=value`. 
That cookie now can **only** be sent to secure URLs: https://site.example. This includes cases where a different site, http://other.example, embeds resources from site.example: http://site.example/otherresource.jpg will **not** get sent the cookie, https://site.example/somethingelse.jpg will get sent the cookie.

#### TL;DR
"Schemeful Same-Site" affects the context cookies will be sent/set in. "Is this URL the same site as the one I'm on if I consider the schemes?"

“Scheme-Bound Cookies” affects the types of schemes the cookies will be sent to. "Does the scheme of the request URL match the scheme of the original, response, URL that set this cookie?"

#### Example of Differences
This example showcases that it's possible to have a situation in which the cookies that "Schemeful Same-Site" and "Scheme-Bound Cookies" each send can differ.

1. A user navigates to https://website.example which contains some embedded resources which want to set and read cookies.

   * A response to the request for https://website.example/setsacookie.jpg has `Set-Cookie: mycookie=val; SameSite=Strict`

2. The user then navigates to http://website.example which is the same page as before, served over an insecure connection.

3. The browser checks to see if it can send the cookie on a request for http://website.example/readsacookie.jpg

   * In the current world (i.e. without either proposal) this cookie is allowed to be sent as the "same-site context" is same-site: only the registrable domain matters and they match.

   * “Schemeful Same-Site” would also allow this cookie to be sent as the "same-site context" is still same-site when considering scheme: the user is on http://website.example and the cookie wants to be sent to http://website.example

   * “Scheme-Bound Cookies” would not allow this cookie to be sent as the scheme isn't the same as the one the cookie was set by: the cookie was set by https and is trying to be sent to http.

4. The browser also checks to see if it can send the cookie on a request for https://website.example/readsacookie2.jpg

   * In the current world the cookie is allowed to be sent as the "same-site context" is same-site: the registrable domains match.

   * “Schemeful Same-Site” would not allow this cookie to be sent as the "same-site context" is now cross-site: the user is on http://website.example and the cookie wants to be sent to https://website.example.

   * “Scheme-Bound Cookies” would allow this cookie to be sent as the scheme is the same as the one the cookie was set by: the cookie was set by https and is trying to be sent to https.

5. Finally the user navigates to a different website, https://othersite.example, which embeddeds https://website.example/readsacookie2.jpg; the browser checks to see if it can send the cookie on the request for that subresource

   * In the current world the cookie not allowed to be sent as the "same-site context" is cross-site: the registrable domains don't match
   
   * “Schemeful Same-Site” would not allow this cookie to be sent as the "same-site context" is cross-site: the registrable domains don't match.
   
   * "Scheme-Bound Cookies" would allow this cookie to be sent as the scheme is the same as the one the cookie was set by: the cookie was set by https and is trying to be sent to https. (It's important to note that even if "Scheme-Bound Cookies" rules allow this cookie to be sent it will still be blocked by the browser due to the same-site rules.)

### Is "Schemeful Same-Site" the same as "Schemeful SameSite"?
Yes. While "Schemeful Same-Site" is the proposal's name, "Schemeful SameSite" is occasionally used. We encourage the "Schemeful Same-Site" spelling.
