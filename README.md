# Schemeful Same-Site Explainer

## Authors
* bingler@chromium.org
* davidben@chromium.org
* kaustubhag@chromium.org

## Introduction
The [SameSite](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.8) cookie attribute is designed to defend against CSRF attacks but currently does not take the scheme of the site into account. This was [originally to assist sites during their transition to https](https://github.com/w3c/webappsec-fetch-metadata/issues/34#issuecomment-527338651), however it results in the secure and insecure versions of the same host being considered same-site. A network attacker could thus impersonate http://site.example and use it to bypass SameSite protections on https://site.example. Between this security flaw and [HTTPS usage markedly increasing](https://transparencyreport.google.com/https/overview), we believe it is time to change this definition.

The web ecosystem is already moving in this direction with the [HTML spec](https://html.spec.whatwg.org/multipage/origin.html#same-site) having been updated with the new definition of same site and the in-progress W3C PING [Privacy Threat Model](https://w3cping.github.io/privacy-threat-model/#terminology) utilizing it.

## Proposal
Modify SameSite’s implementation in the user agent to consider origins with different schemes as cross-site. Thus https://site.example and http://site.example would now be considered cross-site.

Part of this effort will be to update [Incrementally Better Cookies](https://tools.ietf.org/html/draft-west-cookie-incrementalism-00) to match the intended behavior.

## Goal
To protect users from CSRF attacks on secure origins which are carried out by insecure origins with the same domain.

## Non-Goals
This proposal does not address [weak confidentiality](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.5) or [weak integrity](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.6) of cookies, specifically in the context of scheme isolation. This means that both secure and insecure origins will retain access to the same set of cookies. I.e., if a user visits http://site.example with http://site.example/image.jpg and the response sets a cookie then when that user visits https://site.example the request to https://site.example/image.jpg will still send that same cookie.

"[Scheme-Bound Cookies](https://github.com/mikewest/scheming-cookies)" is a proposal aiming to address this issue.

## Recommended Action
Affected sites are encouraged to fully migrate to HTTPS.

## Changed Behavior
“Cross-scheme” in this context means origins which are [schemelessly](https://html.spec.whatwg.org/multipage/origin.html#schemelessly-same-site) same-site but have differing schemes. E.g., http://site.example and https://site.example are considered cross-scheme.

* SameSite=Lax and SameSite=Strict cookies will no longer be sent in cross-scheme subresource requests, i.e. http-to-https and https-to-http (mixed content), regardless of HTTP request method.
* SameSite=Lax and SameSite=Strict cookies can no longer be set by a cross-scheme subresource.
* SameSite=Lax and SameSite=Strict cookies will no longer be sent in cross-scheme top-level POSTs.
* SameSite=Strict cookies will no longer be sent in cross-scheme top-level navigations.

## Examples
Below are two common use cases which will be changing.

### Top-level POST

Cross-scheme top-level POSTS will no longer send SameSite=Lax cookies.

For example, a user writes up a forum post on http://messageboard.example. When the user clicks submit a POST request is sent to https://messageboard.example/submit, however the forum login cookie, which is marked with SameSite=Lax, will not be sent and the user’s post will fail.
### Subresources

Requests to cross-scheme subresources will no longer send SameSite=Lax or SameSite=Strict cookies. Similarly, responses from cross-scheme subresources will no longer be able to set such cookies.

For example, if http://site.example embeds an image from https://site.example, SameSite=Lax and SameSite=Strict cookies will no longer be sent on the request to https://site.example, and any such cookies in the response will be ignored. 

## Questions
### How do Schemeful Same-Site and Scheme-Bound Cookies differ?

"Schemeful Same-Site" and "[Scheme-Bound Cookies](https://github.com/mikewest/scheming-cookies)" are both trying to move cookies closer to an origin based security model; these two proposals complement one another and one is not a subset of the other.

#### Schemeful Same-Site
"Schemeful Same-Site", and the current version of same-site, is concerned with the browsing context in which the cookie will be sent or recieved. The browsing context can be thought of as the answer to the question "Is the request/response url the same site as the one I'm on?" More specifically "Is the request/response url 'same-site' or 'cross-site' with the current site I'm on?" 

In the current world the browser context is determined by checking the registrable domain: http://site1.example is *not* same-site with http://site2.example, but http://site1.example and https://site1.example would be same-site.
With "Schemeful Same-Site" we now consider the scheme along with the registrable domain: http://site1.example is cross-site, not same-site, with http://site2.example, http://site1.example is also cross-site with https://site1.example, but https://site1.example is same-site with https://site1.example. 

#### Scheme-Bound Cookies
"Scheme-Bound Cookies" is concerend with the connection overwhich a cookie is being sent or set.
In the current world a response from https://site.example/resource.jpg could set a cookie: `Set-Cookie: mycookie=value`.
That cookie can then be sent to any url with "site.example" as the host regardless of scheme: http://site.example/otherresource.jpg, https://site.example/somethingelse.jpg, etc.

With "Scheme-Bound Cookies" the browser will no longer send the cookie to a different scheme than the one that set it.
With that in mind: a response from https://site.example/resource.jpg sets a cookie: `Set-Cookie: mycookie=value`. 
That cookie now can **only** be sent to https://site.example urls. This includes cases where a different site, http://other.example, embedded resources from site.example: http://site.example/otherresource.jpg will **not** get sent the cookie, https://site.example/somethingelse.jpg will get sent the cookie.

#### TL;DR
"Schemeful Same-Site" affects the browsing context cookies will be sent/set in. "Is this url the same site as the one I'm on if I consider the schemes?"

“Scheme-Bound Cookies” affects the connections the cookies will be sent on. "Does the scheme of the request url match the scheme of the original response url that set this cookie?"

### Example
This example showcases that it's possible to have a situation in which the cookies that "Schemeful Same-Site" and "Scheme-Bound Cookies" each send can differ.

1. A user navigates to https://website.example which contains some embedded resources which want to set and read cookies.

   * A response to the request for https://website.example/setsacookie.jpg has `Set-Cookie: mycookie=val; SameSite=Strict`

2. The user then navigates to http://website.example which is the same page as before, served over an insecure connection.

3. The browser checks to see if it can send the cookie to http://website.example/readsacookie.jpg

   * In the current world (i.e. without either proposal) this cookie is allowed to be sent as this is a same-site context: only the registrable domain matters and they match.

   * “Schemeful Same-Site” would also allow this cookie to be sent as we're still in a same-site context when considering scheme: the user is on http://website.example and the cookie wants to be sent to http://website.example

   * “Scheme-Bound Cookies” would not allow this cookie to be sent as the connection isn't the same as the one the cookie was set by: the cookie was set on https and is trying to be sent on http.

4. Immediately after the browser checks to see if it can send the cookie to https://website.example/readsacookie2.jpg

   * In the current world the cookie is allowed to be sent as this is a same-site context: the registrable domains match.

   * “Schemeful Same-Site” would not allow this cookie to be sent as we're now in a cross-site context: the user is on http://website.example and the cookie wants to be sent to https://website.example.

   * “Scheme-Bound Cookies” would allow this cookie to be sent as the connection is the same as the one the cookie was set by: the cookie was set on https and is trying to be sent on https.

5. Finally the user navigates to a different website, https://othersite.example, which embeddeds https://website.example/readsacookie2.jpg

   * In the current world the cookie not allowed to be sent as this is a cross-site context: the registrable domains don't match
   
   * “Schemeful Same-Site” would not allow this cookie to be sent as this is a cross-site context: the registrable domains don't match.
   
   * "Scheme-Bound Cookies" would allow this cookie to be sent the connection is the same as the one the cookie was set by: the cookie was set on https and is trying to be sent on https.



