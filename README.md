# Schemefel Same-Site Explainer

## Authors
* bingler@chromium.org
* davidben@chromium.org
* kaustubhag@chromium.org

## Introduction
The SameSite cookie attribute is designed to defend against CSRF attacks but currently does not take the scheme of the site into account. This was originally to assist sites during their transition to https, however it results in the secure and insecure versions of the same host being considered same-site. A network attacker could thus impersonate http://site.example and use it to bypass SameSite protections on https://site.example. Between this security flaw and HTTPS usage markedly increasing, we believe it is time to change this definition.

The web ecosystem is already moving in this direction with the HTML spec having been updated with the new definition of same site and the in-progress W3C PING Privacy Threat Model utilizing it.
