%%%
title = "Efficient RDAP Referrals"
abbrev = "Efficient RDAP Referrals"
ipr = "trust200902"
area = "Internet"
workgroup = "Registration Protocols Extensions (regext)"
consensus = true

[seriesInfo]
name = "Internet-Draft"
value = "draft-ietf-regext-rdap-referrals-00"
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
be referred to a related RDAP record for a resource.

{mainmatter}

# Introduction

Many Registration Data Access Protocol (RDAP, described in [@!RFC7480],
[@!RFC7481], [@!RFC9082], [@!RFC9083] and others) resources contain links to
related RDAP resources.

For example, in the domain space, an RDAP record for a domain name received from
the registry operator may include a link for the RDAP record for the same
object provided by the sponsoring registrar, while in the IP address space, an
RDAP record for an address allocation may include links to enclosing or
sibling prefixes.

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
an RDAP server refer them to the URL of a related resource.

# RDAP Referral Request

To request a referral to a related resource, the client sends an HTTP `GET`
request to the RDAP server with a path of the form:

```
/referrals0/<relation>/<lookup path>
```

The client replaces `<lookup path>` with the lookup path of the object being sought
and the `<relation>` with the desired relationship type.
For example, a referral query for the domain example.com would be:

```
/referrals0/related/domain/example.com
```

The referral query for the parent network of `192.0.2.42` would have the following
full path:

```
/referrals0/rdap-up/ip/192.0.2.42
```

Lookup paths for domain names, IP networks, autonomous system numbers, nameservers, and
entities are described in [@!RFC9082]. Lookups defined by RDAP extensions may also
use this extension.

Referral requests for searches, where more than one object is returned, as described by
[@!RFC9083] are not supported. Servers MUST return an HTTP 400 for these requests.

# RDAP Referral Response

If the object specified in the request exists, then the response **MUST** have an
HTTP status code of 300. The response body **MUST** be a minimal RDAP response
(as described in [@!RFC9083]) for the object in the response, containing only
the `objectClassName` and `links` properties. The client may then select the
appropriate link itself, based on the link properties, or present them to the
user for review.

An example for the query "https://rdap.nic.example/referrals0/related/domain/example.com":

```
HTTP/2 300
Content-Type: application/rdap+json
Access-Control-Allow-Origin: *
Vary: Accept

{
  "objectClassName": "domain",
  "links": [
    {
      "value": "https://rdap.nic.example/domain/example.com",
      "rel": "related",
      "href": "https://rdap.example.com/domain/example.com",
      "type": "application/rdap+json"
    }
  ]
}
```

In scenarios where an object has multiple related links, a server MAY return all
links of the requested relation, but the client will most likely only use one of them.
That is, the client is under no obligation to process more than one link.

Servers MAY include a "self" link as specified by [@!RFC9083] which will enable
client caching of objects. However, when using a "self" link with this extension,
the `href` property MUST be the request URI, not the object URI, and clients MUST
use the `href` property for object caching to prevent cache collisions of full
objects with these responses.

In all cases, the `value` property of the link objects in the response **MUST** be
the URI of the object, not the request URI, since the `value` property specifies
the context URI of the link.

An example with multiple related links and a self link:

```
HTTP/2 300
Content-Type: application/rdap+json
Access-Control-Allow-Origin: *
Vary: Accept

{
  "objectClassName": "domain",
  "links": [
    {
      "value": "https://rdap.nic.example/domain/example.com",
      "rel": "related",
      "href": "https://rdap.example.com/domain/example.com",
      "type": "application/rdap+json"
    },
    {
      "value": "https://rdap.nic.example/domain/example.com",
      "rel": "related",
      "href": "https://rdap.example.com/domain/example.com",
      "type": "application/rdap+json"
    },
    {
      "value": "https://rdap.nic.example/related/domain/example.com",
      "rel": "self",
      "href": "https://rdap.nic.example/related/domain/example.com",
      "type": "application/rdap+json"
    }
  ]
}
```

## Caching by Intermediaries

To facilitate caching of RDAP resources by intermediary proxies, servers which
provide a referral based on the value of the `Accept` header field in the
request **MUST** include a `Vary` header field (See Section 12.5.5 of
[@!RFC2535]) in the response. This field **MUST** include `accept` and **MAY**
include other header field names.

Example:

```
Vary: accept, accept-language
```

# RDAP Conformance

Servers which implement this specification **MUST** include the string
"`referrals0`" in the "`rdapConformance`" array in all RDAP
responses.

# IANA Considerations

IANA is requested to register the following value in the RDAP Extensions
Registry:

**Extension identifier:** `referrals0`

**Registry operator:** any.

**Published specification:** this document.

**Contact:** the authors of this document.

**Intended usage:** this extension allows clients to request to be referred to a
related resource for an RDAP resource.

# Change Log

This section is to be removed before publishing as an RFC.

## Changes from 00 to 01

* Switch to using a path segment and 30x redirect.

## Changes from draft-brown-rdap-referrals-02 to draft-ietf-regext-rdap-referrals-00

* Nothing apart from the name.

## Changes from 01 to 02

* add this change log.

## Changes from 00 to 01

* change extension identifer from `registrar_link_header` to `referrals0`.

{backmatter}
