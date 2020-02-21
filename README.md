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
This proposal does not address [weak confidentiality](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.5) or [weak integrity](https://tools.ietf.org/html/draft-ietf-httpbis-rfc6265bis-05#section-8.6) of cookies, specifically in the context of scheme isolation. This means that both secure and insecure origins will retain access to the same set of cookies. I.e., if http://site.example sets a cookie then https://site.example can still read that cookie.

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

