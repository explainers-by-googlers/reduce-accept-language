# Explainer: Reduce fingerprinting in the Accept-Language header

This proposal is an early design sketch by Privacy Sandbox to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [A Problem](#a-problem)
- [A Proposal](#a-proposal)
  - [For example...](#for-example)
  - [Language negotiation](#language-negotiation)
  - [No `Content-Language` in the response header](#no-content-language-in-the-response-header)
  - [`Content-Language` matches `Accept-Language`](#content-language-matches-accept-language)
  - [`Content-Language` doesn't match `Accept-Language`](#content-language-doesnt-match-accept-language)
- [Feature Compatibility](#feature-compatibility)
- [Challenges](#challenges)
  - [Web Compatibility](#web-compatibility)
  - [Large Avail-Language size](#large-avail-language-size)
- [FAQ](#faq)
  - [What user needs are solved by reducing the Accept-Language header?](#what-user-needs-are-solved-by-reducing-the-accept-language-header)
  - [Do we need to update the JavaScript interface (i.e. Navigator.languages) too?](#do-we-need-to-update-the-javascript-interface-ie-navigatorlanguages-too)
  - [How to prevent malicious sites from learning users' preferred languages?](#how-to-prevent-malicious-sites-from-learning-users-preferred-languages)
  - [What about the quality value in the Accept-Language header?](#what-about-the-quality-value-in-the-accept-language-header)
  - [Does resending the language negotiation request increase latency when the Content-Language doesn’t match any of the user's accepted languages?](#does-resending-the-language-negotiation-request-increase-latency-when-the-content-language-doesnt-match-any-of-the-users-accepted-languages)
- [Considered alternatives](#considered-alternatives)
  - [Moving the language header to a new client hint `Sec-CH-Lang`](#moving-the-language-header-to-a-new-client-hint-sec-ch-lang)
  - [Sending nothing on the first request](#sending-nothing-on-the-first-request)
- [Reference](#reference)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## A Problem

Chrome (and other browsers) send all of the user's language preferences on every HTTP request via the `Accept-Language` header.  The header's value contains a lot of entropy about the user that is sent to servers by default.  While some sites use this information for content negotiation, servers can also [passively](https://w3c.github.io/fingerprinting-guidance/#dfn-passive-fingerprinting) capture this information without the user's awareness to fingerprint a user.  As part of the Chrome team’s anti-covert tracking efforts, we would like to improve privacy protections by minimizing passive fingerprinting surfaces.

An example of the `Accept-Language` header sent on a request:

```
GET /foo HTTP/1.1
Host: example.com
Accept-Language: en-US,en;q=0.9,zh-CN;q=0.8,zh;q=0.7
```

Our goal is to find the best possible language between the user's preferred languages and the websites supported languages. Since servers usually don't need to know the full list of languages the user accepts during [content negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation), this explainer proposes a mechanism to update the `Accept-Language` behavior which would allow us to better protect user privacy while still allowing sites to perform language-based content negotiation.

## A Proposal

We propose that, by default, the browser should only send the user's most preferred language in the `Accept-Language` header instead of sending all languages. That means we only send only one language in the `Accept-Language` request header.

### For example...

As proposed, users will only send a single language in the `Accept-Language` request header, and the site will respond with the document language in the `Content-Language` response header. For example, an Accept-Language's initial request to `https://example.com` will include the following request headers:

```
Get / HTTP/1.1
Host: example.com
Accept-Language: en
```

The server will respond with a `Content-Language` header indicating the document language.

```
HTTP/1.1 200 OK
Content-Language: en
```

### Language negotiation

Since we will only send a single language in the `Accept-Language` header, we will cover the cases which will require further language negotiation to deliver the content in the best possible language for the user.

### No `Content-Language` in the response header

This means the content is intended for all language audiences. No action should be taken by the browser. Users will receive whatever content the sites provide.

### `Content-Language` matches `Accept-Language`

In this case, the user received content in their preferred language. No further language negotiation is needed.

### `Content-Language` doesn't match `Accept-Language`

Instead of sending a full list of the users' preferred languages from browsers and letting sites figure out which language to use, we propose a language negotiation process in the browser, which means in addition to the `Content-Language` header, the site also needs to respond with a header indicating all languages it supports using the `Avail-Language` header (as proposed in [HTTP Availability Hints](https://mnot.github.io/I-D/draft-nottingham-http-availability-hints.html#section-5.3)).

For now, the response header will include both `Content-Language` and `Avail-Language`. Let’s see how language negotiation works with the following examples:


1. Site supported language contains user’s preferred language.
* Suppose the user prefers English (en) and they also accept Spanish (es), and they visit `https://example.com`. The following request header `Accept-Language` provides user's preferred language:

    ```
    GET / HTTP/1.1
    Host: example.com
    Accept-Language: en
    ```

* `https://example.com` doesn’t have a page in English. The server either guesses the users’ preference or sends a page with a default language such as French to users.  To allow language negotiation in the browser if the language the site chose to send the content in differs from the user’s preferred language, the site will respond with a list of supported languages in the Avail-Language header: Spanish (es) and French (fr).

    ```
    HTTP/1.1 200 OK
    Content-Language: fr
    Vary: Accept-Language   # Vary is used to key the HTTP cache
    Avail-Language: es, fr
    ```

* The browser sees that the `Content-Language` doesn’t match any of the user's accepted languages, but it knows the server can provide the content in one of the user's accepted languages: Spanish (es).  Therefore, the browser resends the request as follows (the browser will only retry once for each request to avoid infinite retries):

    ```
    GET / HTTP/1.1
    Host: example.com
    Accept-Language: es
    ```

* Now, the site will respond with a page in Spanish (es) which matches one of the user's accepted languages.

    ```
    HTTP/1.1 200 OK
    Content-Language: es
    Vary: Accept-Language   # Vary is used to key the HTTP cache
    Avail-Language: es, fr
    ```

2. The site’s supported languages don’t match any of the user’s accepted languages

    In this case, we keep the existing behavior, users will receive whatever content the site provides. One option for users is that they can switch to using the translation service to translate the page to a language they prefer.

## Feature Compatibility

While we propose to limit the amount of language information sent to a site, our proposal ensures that from the site’s point of view, there are no regressions in behavior with respect to delivering the right content to the user.  This proposal allows a site to maintain all language-based functionality, without having to receive all the user’s language preferences, through the client-side language selection mechanism described above.

## Challenges

This section attempts to document the major challenges for the proposal to gain ecosystem adoption.

### Web Compatibility

In addition to the scenarios where we reduce the fingerprinting to protect user’s privacy, we also want to maintain web compatibility. Even though sending the preferred language by default might reveal some information about the user, we feel it’s important to minimize the breakage of features that depend on `Accept-Language` as much as possible to maintain stability of the web ecosystem.

### Large Avail-Language size

One of the challenging situations is that a site can support infinite [ possible languages](https://www.iso.org/iso-639-language-codes.html) because of arbitrary subtags, and the `Avail-Language` header could get large, however unlikely. It could be problematic to send such a large field in the HTTP response header. One option to mitigate this issue is that servers can simply not include the Avail-Language header in the server's response when a site supports all or most of the possible languages. Another possible option is that we might suggest sites send `Avail-Language: *` to indicate sites that support all or most of the possible languages. In addition to those, we could also send all the languages sites supported in the `Avail-Language` header regardless of how large the payload will be. Assuming about 500 languages with 5 characters on average per language/locale, the total size of the Avail-Language header would be about 2.5KB.

## FAQ

### What user needs are solved by reducing the Accept-Language header?

As we mentioned at the beginning of this explainer, we believe that the `Accept-Language` header is a source of passive entropy. By only sending a single language in the `Accept-Language` header, it helps Chrome to protect users' privacy.

### Do we need to update the JavaScript interface (i.e. Navigator.languages) too?

Developers usually access two JavaScript API related to language: `navigator.language` and `navigator.languages`. We can keep `navigator.language` as it is for now since it returns the most preferred language of the user. For `navigator.languages`, we could limit it to return one language (as an array) which would be similar to how `navigator.language` works.

### How to prevent malicious sites from learning users' preferred languages?

Malicious sites can learn a full list of users' preferred languages by continuously providing different languages in the response. To address this, davidben@ propose some [possibilities](https://github.com/davidben/client-language-selection#abuse). We can start with rate limit language changes or distinct language lists for a site. To support our initial thoughts we need to gather metrics to understand how this plays in the ecosystem.

### What about the quality value in the Accept-Language header?

The `Accept-Language` header includes a quality value (q-factor weighting, e.g. `q=0.9` in the example below) to indicate relative preference.

```
Accept-Language: en-US,en;q=0.9,de;q=0.8
```

As the [RFC4647](https://datatracker.ietf.org/doc/html/rfc4647#section-2.3) shows, the default value of `q=1` is used if there is no quality value specified. In this case, this proposal plans to remove `q` weighting unless we figure out it’s required for web compatibility. In which case q weighting will be frozen to `q=1`.

### Does resending the language negotiation request increase latency when the Content-Language doesn’t match any of the user's accepted languages?

As mentioned above, the browser will only retry once for each request to resend the request to do language negotiation. We will keep monitoring the benchmarking metrics when experimenting to see whether it would increase latency significantly.

## Considered alternatives

### Moving the language header to a new client hint `Sec-CH-Lang`

The [lang client hint RFC](https://tools.ietf.org/id/draft-west-lang-client-hint-00.txt) draft defines a client hint to allow developers to opt-in perform content negotiation based on the users’ preferred languages. However, davidben@ elaborates the reason why we prefer `Accept-Language` in [Client-side Language Selection proposal](https://github.com/davidben/client-language-selection#considered-alternatives).


### Sending nothing on the first request

Another [proposal](https://github.com/WICG/lang-client-hint) suggests not sending the `Accept-Language` header in the initial request. However, we believe that it would be better to send the `Accept-Language` header in the initial request to prevent the additional cost of extra round trip sending a retry with the correct language.

Some internal Google research (and intuition) indicates the user's top most preferred language has a strong correlation with the rough geolocation, so the header leaks a limited amount of additional entropy to all sites when we send the most preferred language only.  In order to prevent a poor user experience where users get a site in a language that they can't understand on the first request, we believe it would be better to send the `Accept-Language` header in the initial request with just the most preferred language to prevent an extra round-trip for the sake of content negotiation.  Also, the [Client-side Language Detection proposal](https://github.com/davidben/client-language-selection#client-hints) outlines the reasons why we prefer using the `Accept-Language` header instead of introducing a new client hint.

Sending the primary language in the `Accept-Language` header will allow us to also gather data about the effectiveness of this approach, which we can use to inform decisions about subsequent steps to further reduce the entropy in the `Accept-Language` header.

## Reference

* [https://github.com/davidben/client-language-selection](https://github.com/davidben/client-language-selection)
* [https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation)
* [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language)
* [https://wonderproxy.com/blog/accept-language/](https://wonderproxy.com/blog/accept-language/)
* https://github.com/WICG/lang-client-hint
* https://www.rfc-editor.org/rfc/authors/rfc9110.html#name-accept-language
