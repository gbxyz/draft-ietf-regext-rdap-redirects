%%%
title = "Explicit RDAP Redirects"
abbrev = "Explicit RDAP Redirects"
ipr = "trust200902"
area = "Internet"
workgroup = "Registration Protocols Extensions (regext)"
consensus = true

[seriesInfo]
name = "Internet-Draft"
value = "draft-ietf-regext-rdap-redirects-00"
stream = "IETF"
status = "standard"

[[author]]
fullname="Gavin Brown"
organization = "ICANN"
  [author.address]
  email = "gavin.brown@icann.org"
  uri = "https://icann.org"
  [author.address.postal]
  street = "12025 Waterfront Drive, Suite 300"
  city = "Los Angeles"
  region = "CA"
  code = "90292"
  country = "US"

[[author]]
fullname="Andy Newton"
organization = "ICANN"
  [author.address]
  email = "andy.newton@icann.org"
  uri = "https://icann.org"
  [author.address.postal]
  street = "12025 Waterfront Drive, Suite 300"
  city = "Los Angeles"
  region = "CA"
  code = "90292"
  country = "US"
%%%

.# Abstract

This document describes an RDAP extension that allows RDAP clients to request to
be redirected to a related RDAP record for a resource.

{mainmatter}

# Introduction

Many Registration Data Access Protocol (RDAP, described in [@!RFC7480],
[@!RFC7481], [@!RFC9082], [@!RFC9083] and others) resources contain links to
related RDAP resources.

For example, in the domain space, an RDAP record for a domain name received from
the registry operator may include a link for the RDAP record for the same
object provided by the sponsoring registrar (for example, see
[@gtld-rdap-profile]), while in the IP address space, an RDAP record for an
address allocation may include links to enclosing or sibling prefixes.

In both cases, RDAP service users are often equally if not more interested in
these related RDAP resources than the resource provided by the TLD registry or
RIR.

While RDAP supports redirection of RDAP requests using HTTP redirections (which
use a `3xx` HTTP status and the "`Location`" header field, see Section 15.4 of
[@!RFC9110]), it is not possible for RDAP servers to know _a priori_ whether a
client requesting an RDAP record is doing so because it wants to retrieve a
related RDAP record, or its own, so it can only respond by providing the full
RDAP response. The client must then parse that response in order to extract the
relevant URL from the "`links`" property of the object.

This results in the wasteful expenditure of time, compute resources and
bandwidth on the part of both the client and server.

This document describes an extension to RDAP that allows clients to request that
an RDAP server redirect them to the URL of a related resource.

# RDAP Redirect Request

To request a redirect to a related resource, the client sends an HTTP `GET`
request with a URL of the form:

```
<base URL>redirects0_ref/<relation>/<lookup path>
```

The client replaces `<base URL>` with the applicable base URL (which, as per
[@RFC9224], has a trailing `/` character), `<relation>` with the desired
relationship type, and `<lookup path>` with the lookup path of the object being
sought (which, as per [@RFC9082], does not have a leading `/` character).

For example, the URL of a redirect query for the domain `example.com`, where the
base URL for the "`com`" TLD is `https://rdap.example.com/rdap/`, would be:

```
https://rdap.example.com/rdap/redirects0_ref/related/domain/example.com
```

The redirect query for the parent network of `192.0.2.42` with the base URL of
`https://rdap.example.net/` would be:

```
https://rdap.exampple.net/redirects0_ref/rdap-up/ip/192.0.2.42
```

Lookup paths for domain names, IP networks, autonomous system numbers,
nameservers, and entities are described in [@!RFC9082]. Lookups defined by RDAP
extensions may also use this extension.

Redirect requests for searches, where more than one object is returned, and help
queries, as described by [@!RFC9083], are not supported. Servers MUST return an
HTTP 400 for these requests.

# RDAP Redirect Response

If the object specified in the request exists, a single appropriate link exists,
and the client is authorised to perform the request, the server response
**MUST**:

1. have an HTTP status code of 301 (Moved Permanently), 302 (Found), 303 (See
   Other), 307 (Temporary Redirect) or 308 (Permanent Redirect, see Section
   15.4.9 of [@!RFC9110]); and

2. include an HTTP `Location` header field, whose value contains the URL of the
   linked resource.

If the server cannot find an appropriate link, the response **MUST** have an
HTTP status of 404.

If an RDAP server holds in its datastore more than one relationship type for
an object, a scenario that is possible but not common, only one of the URLs, as
determined by server policy, can be returned.

The following examples use the HTTP/1.1 message exchange syntax as seen in
[@!RFC9110].

An example of a redirect request from a domain registry to a domain registrar:

```
Client Request:

GET /redirects0_ref/related/domain/example.com HTTP/1.1
Accept: application/rdap+json

Server Response:

HTTP/1.1 307 Temporary Redirect
Location: https://registrar.example/domain/example.com
```

An example of a redirect request for a parent IPv4 network:

```
Client Request:

GET /redirects0_ref/rdap-up/ip/192.0.2.42 HTTP/1.1
Accept: application/rdap+json

Server Response:

HTTP/1.1 307 Temporary Redirect
Location: https://rir.example/ip/192.0.2.0/24
```

An example of a redirect request for a parent IPv6 network:

```
Client Request:

GET /redirects0_ref/rdap-up/ip/2001%3adb8%3a%3a1 HTTP/1.1
Accept: application/rdap+json"

Server Response:

HTTP/1.1 307 Temporary Redirect
Location: https://rir.example/ip/2001%3adb8%3a%3a/32
```

## Selecting The Appropriate Link

When the server receives a redirect request, it must select which of an object's
links it should use to construct the response.

The `rel` property of the selected link **MUST** match `<relation>` path
segment of the request. The `type` and `hreflang` properties of the link, if
present, **MUST** match the `Accept` and (if specified) `Accept-Language` header
fields of the request.

## Caching by Intermediaries

To facilitate caching of RDAP resources by intermediary proxies, servers which
provide a redirect based on the value of the `Accept` header field in the
request **MUST** include a `Vary` header field (See Section 12.5.5 of
[@!RFC9110]) in the response. This field **MUST** include `accept`, and **MAY**
include other header field names.

Example:

```
Vary: accept, accept-language
```

## Client Processing of Redirect Responses

Note that as per Section 10.2.2 of [!@RFC9110], the URI-reference in `location`
header fields **MAY** be relative. For relative references, RDAP clients
**MUST** compute the full URI using the request URI.

# RDAP Conformance {#rdapConformance}

Servers which implement this specification **MUST** include the string
"`redirects0`" in the "`rdapConformance`" array in responses to RDAP "help"
queries.

# Bootstrap Use Case {#bootstrap}

The primary use case of this extension is a one-hop redirect, where the client is not interested in the use
of this extension beyond the first redirect. An example use case is querying a bootstrap redirect server for
the authoritative server of information according to the IANA RDAP bootstrap information. 

```
Client Request:

GET /redirects0_ref/rdap-bootstrap/ip/2001%3adb8%3a%3a1 HTTP/1.1
Accept: application/rdap+json"

Server Response:

HTTP/1.1 307 Temporary Redirect
Location: https://rir1.example/ip/2001%3adb8%3a%3a/32
```

Other uses cases may exist, but for this specific use case, this document registers the "rdap-bootstrap"
link relationship type.

# Multi-Hop Redirect Limitations

In some scenarios, a target server might have a policy to issue another redirect using this extension.
For example:

```
Client Request to rir1.example:

GET /redirects0_ref/rdap-top/ip/2001%3adb8%3a%3a1 HTTP/1.1
Accept: application/rdap+json"

Server Response:

HTTP/1.1 307 Temporary Redirect
Location: https://rir2.example/redirects0_ref/rdap-top/ip/2001%3adb8%3a%3a/32
```

In this scenario rir1.example is redirecting to rir2.example with a "/redirects0_ref" path. However,
not all servers may support this extension. Therefore, the "/redirects0_ref" path defined in this
specification MUST only be used in an HTTP redirect if the server issuing the redirect is assured that the
target server of the redirect supports this extension.

Furthermore, servers SHOULD only use the "/redirects0_ref" path in an HTTP redirect when the link relationship
type is one for a terminal relationship such as "rdap-top" and "rdap-bottom" (i.e., "rdap-up" and "rdap-down"
do not explicitly express a relationship that is the end of a series of redirects).

# IANA Considerations

## RDAP Extension Identifier

IANA is requested to register the following value in the [@rdap-extensions]
Registry:

**Extension identifier:** `redirects0`

**Registry operator:** any.

**Published specification:** this document.

**Contact:** the authors of this document.

**Intended usage:** this extension allows clients to request to be redirected to a
related resource for an RDAP resource.

## Link Relations {#linkrelation}

IANA is requested to register the following value into the [@link-relations] registry:

Relation Name: rdap-bootstrap
Description:
Refers to an RDAP parent object in a hierarchy of objects.
Reference:
This document once published as an RFC.

# Security Considerations

A malicious HTTP redirect has the potential to create an infinite loop, which
can exhaust resources on both client and server side.

To prevent such loops, RDAP servers which receive redirect requests for the
`self` relation **MUST** respond with a 400 HTTP status.

As described in Section 15.4 of [!@RFC9110], when processing server responses,
RDAP clients **SHOULD** detect and intervene in cyclical redirections.

# Change Log

This section is to be removed before publishing as an RFC.

## Changes from draft-ietf-regext-rdap-referrals-02 to draft-ietf-regext-rdap-redirects-00

* Consistely refer to "redirect" instead of "referral". This includes changing
  the extension identifier to `redirects0` and the document name.

* Added {#bootstrap} and {#linkrelation}.

* Correct specification of the redirect query path.

* Updated (#rdapConformance) to limit the use of the extension identifier to
  help responses.

* Include 308 in the list of redirection HTTP status codes.

Thanks to Jasdip Singh for identifying these issues.

## Changes from 01 to 02

* Add reference to [@gtld-rdap-profile] which describes how gTLD RDAP servers
  link to registrar RDAP resoures.

* Include `<base path>` in the path specification, and remove the `/` between
  `<relation>` and `<lookup path>` so that naive URL construction works.

* Reuse the language from RFC 7480 on HTTP status codes used for redirection.

* Fix HTTP status code in the examples.

* Described the risk of redirection loops and things clients and servers have to
  do.

## Changes from 00 to 01

* Switch to using a path segment and a 30x redirect.

* Describe how the server behaves when multiple links exist.

## Changes from draft-brown-rdap-referrals-02 to draft-ietf-regext-rdap-referrals-00

* Nothing apart from the name.

## Changes from 01 to 02

* add this change log.

## Changes from 00 to 01

* change extension identifer from `registrar_link_header` to `redirects0`.

{backmatter}

<reference anchor="gtld-rdap-profile" target="https://www.icann.org/gtld-rdap-profile">
    <front>
        <title>gTLD RDAP Profile</title>
        <author>
            <organization>ICANN</organization>
        </author>
        <date year="2024"/>
    </front>
</reference>

<reference anchor='link-relations' target='https://www.iana.org/assignments/link-relations/link-relations.xhtml'>
    <front>
        <title>Link Relations</title>
        <author>
            <organization>IANA</organization>
        </author>
    </front>
</reference>

<reference anchor='rdap-extensions' target='https://www.iana.org/assignments/rdap-extensions/rdap-extensions.xhtml'>
    <front>
        <title>RDAP Extensions</title>
        <author>
            <organization>IANA</organization>
        </author>
    </front>
</reference>
