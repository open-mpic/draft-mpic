---
title: "Multi-Perspective Issuance Corroboration (MPIC) Service "
abbrev: "MPIC"
category: info

docname: draft-westerbaan-alldispatch-mpic-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Security Dispatch"
keyword:
 - dcv
 - mpic
venue:
  group: "All Dispatch"
  type: "Working Group"
  mail: "alldispatch@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/alldispatch/"
  github: "open-mpic/draft-mpic"
  latest: "https://open-mpic.github.io/draft-mpic/draft-westerbaan-alldispatch-mpic.html"

author:
 -
    fullname: "Syed Suleman Ahmad"
    organization: Cloudflare
    email: "suleman@cloudflare.com"
 -
    fullname: "Bas Westerbaan"
    organization: Cloudflare
    email: "bas@cloudflare.com"
 -
    fullname: "Henry Birge-Lee"
    organization: Princeton University
    email: "birgelee@princeton.edu"

normative:
 RFC8555:

informative:
 RFC6570:


--- abstract

This memo defines an API for Multi-Perspective Issuance Corroboration (MPIC) services to facilitate domain control validation (DCV) from multiple network perspectives. MPIC enhances the security of publicly-trusted certificate issuance by mitigating the risk of localized, equally-specific BGP hijacking attacks that can undermine traditional DCV methods permitted by the CA/Browser Forum Baseline Requirements for TLS Server Certificates. This API enables Certification Authorities (CAs) to more reliably integrate with external MPIC providers, promoting a more robust and resilient Web PKI ecosystem. The API design prioritizes flexibility, scalability, and interoperability, allowing for diverse implementations and deployment models. This standardization effort is driven by the need to consistently address vulnerabilities in the domain validation process highlighted by recent research and real-world attacks, as reflected in Ballot SC-067 V3 of the CA/Browser Forum's Server Certificate Working Group.

--- middle

# Introduction

## DCV
Before issuing a certificate to a subscriber,
certificate authorities are required to validate that the subscriber
indeed controls the domains on the certificate.
For this purpose certificate authorities use various methods
of domain control validation (DCV), including
but not limited to via HTTP, DNS, and ALPN.

## MPIC
Several, but not all, CAs use the specific DCV methods of ACME {{RFC8555}}.
Domain control validation is vulnerable to DNS and BGP hijacks.
These can be partially mitigated by performing DCV from multiple
network perspectives, which is dubbed "multiple perspective issuance corroboration" (MPIC).
corroboration (MPIC) for domain control validation.
Ballot SC-67 v3 of CA/B forum requires MPIC to be performed
by all certification authorities (CAs) in the future.

## MPIC service
Running MPIC requires maintaining a presence across the globe.
For smaller CAs it may make sense to run a shared MPIC service
or outsource it to a third party.
This memo specifies a standardised API for such a usecase.
Another usecase is for a CA to have a standby MPIC service
in case its primary fails.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# API Structure

The MPIC API implements a system for domain validation and CAA record checking
using multiple perspectives across different regions.

An MPIC service can accept JSON requests over an arbitrary communication channel with an MPIC client. The MPIC service then produces MPIC responses which are sent back to the MPIC client over the communication channel. The communication channel may be synchronous (i.e., stall open on each MPIC request until an MPIC response is generated) or asynchronous (i.e., send a message to the MPIC service, close, and then receive a future message from the MPIC service containing a corresponding MPIC response). The MPIC service protocol does not provide any form of matching between MPIC requests and MPIC responses. A communication channel is responsible for ensuring that a client can match requests with corresponding responses.
The communication channel MUST provide confidentiality and integrity as well as support for authentication of the MPIC service.

This document describes communication with an MPIC service using HTTPS POST as the communication channel. Alternate communication channels include gRPC, Apache Kafka, RabbitMQ, etc...

A MPIC service running over the HTTPS POST communication channel is identified by a HTTPS url.
As a running example, say `https://mpc.example.com/staging`.

A client requests a MPIC validation from the service
by sending a POST request to the resource `/mpic/draft-00`
below the service URL.
In the running example `https://mpc.example.com/staging/mpic/draft-00`.

[[ The final version of the API will use `/mpic/v1`. Incompatible
   versions of the draft will bump the `-00`. ]]

The body of the HTTP POST is a JSON object that describs the MPIC request.
The service will respond with a JSON object containing MPIC results.

There are three different MPIC validation methods, described below. The request
object has a `method` field that allows to distinguish between each.

## `caa` validation method {#caa-api}

A `caa` requests asks the MPIC service to retrieve the relevant CAA DNS
records for a given domain from multiple perspectives.

This method has the following specific fields.

* `domain` (required, string): The domain to check the CAA records for.

An example request is given below.

~~~
POST /staging/mpic/draft-00
Host: mpc.example.com
Content-Type: application/json

{
 "method": "caa",
 "domain": "some.example.com"
}
~~~

If successful (described in TODO REF below), the response object contains a
`success` field set to `true`, and an `caa` field, which itself is an object
with two fields:

* `domain` The domain on which the CAA records were found. This could be
  a parent domain of the requested `domain`.

* `records` A list of base64 encoded CAA records.

An example of a response for a succesful validation.

~~~
{
 "success": true,
 "caa": {
  "domain": "example.com",
  "records": ["AAVpc3N1ZWxldHNlbmNyeXB0Lm9yZw=="]
 },
 "perspectives": {
  "jfk": {"success": true},
  "fra":  {"success": true},
  "lis": {"success": true}
 }
}
~~~

On failure, the response object will have the `success` field set to `false`,
and an `error` field describing the error.

[[ TODO do we to define the possible errors, or at least assign
        some codes? ]]

An example of a response for an unsuccesful validation.

~~~
{
 "success": false,
  "perspectives": {
  "jfk": {"success": true},
  "fra":  {"success": true},
  "lis": {"success": false}
 },
 "error": "LIS saw record 'xyz' on example.com which was not present from perspective LIS"
}
~~~

[[ TODO How much information to return on error to help debug, and how
   structured should it be? I'd say it's good to be helpful, but it's bad
   to be structured as it's less readable. ]]


## `http-acme` method
`http-acme` requests the MPIC server to perform ACME http-01 challenge validation
{{RFC8555}} for the domain's HTTP server from each distributed perspectives.

Performs a GET from multiple perspectives, and checks whether the body matches
expectation. Optionally, it allows performing an additional CAA record lookup
for the domain.

The request JSON object has the following specific fields.

* `domain_or_ip` (required, string): The domain name or IP address being verified.
* `token` (required, string): The token value defined in {{RFC8555}}  Section 8.3.
* `key_authorization` (required, string): The Key Authorization defined in {{RFC8555}}  Section 8.1.
* `caa_check` (optional, boolean): Performs CAA validation at the same time for the domain as described above. Defaults to true unless `domain_or_ip` is an IP address.

~~~
POST /staging/mpic/draft-00
Host: mpc.example.com
Content-Type: application/json

{
 "method": "http-acme",
 "domain_or_ip": "some.example.com",
 "token": "base64_url_token",
 "key_authorization": "base64_url_token.base64url_Thumbprint_accountKey"
 "caa_check": false,
}
~~~

The MPIC server constructs a URL by populating the URL template
{{RFC6570}}, `http://{domain_or_ip}/.well-known/acme-challenge/{token}`, and verifies that the resulting URL is
well-formed, before making a HTTP GET request to the URL from each vantage
point. Each perspective SHOULD follow redirects when dereferencing the URL.
The MPIC server verifies that the `key_authorization` value provided by the client matches
with the body of the response received from each perspective.

If the above verifications succeeds, then the validation is successful. If
the request fails, or the body does not pass these checks, then it has failed.

Along side, the MPIC server queries for the CAA records for the
`domain_or_ip` if the `caa_check` request parameter is set to "true".

If either HTTP or CAA validation (when requested) fail,
the response objects contains a top-level
`success` field set to `false`, and contains an `error` field
that describes the error.

If both succeed, the response object contains a top-level `success` field set to `true`.

The response also contains an object (located under the key `perspectives`) with keys that uniquiely identify the perspectives used in the reqest.
Each perspective is associated with an object that contains `success` key pointing to a boolean value of `true` or `false` to indicate whether validation was successful at that perspective or not.

If a CAA check was requested, the response object will contain a top
level `caa` field as described in {{caa-api}}.

An example of a response for a succesful validation with `caa-check` set to
false.

~~~
{
 "success": true,
 "perspectives": {
  "jfk": {"success": true},
  "fra":  {"success": true},
  "lis": {"success": true}
 }
}
~~~

An example of a response for a succesful validation with `caa-check` set to
true.

~~~
{
 "success": true,
 "perspectives": {
  "jfk": {"success": true},
  "fra":  {"success": true},
  "lis": {"success": true}
 }
 "caa": {
   "domain": "example.com",
   "records": ["AAVpc3N1ZWxldHNlbmNyeXB0Lm9yZw=="]
 }
}
~~~

An example of a response where the validation failed with caa_check set
to true, and the `caa` method being successful.

~~~
{
 "success": false,
 "perspectives": {
  "jfk": {"success": true},
  "fra":  {"success": true},
  "lis": {"success": false}
 }
 "error": "HTTP found unexpected value at LIS perspective",
}
~~~

Similarly example of a response where caa_check set to true, and both
methods fail.

{
 "success": false,
 "perspectives": {
  "jfk": {"success": false},
  "fra":  {"success": true},
  "lis": {"success": false}
 }
 "caa": {
   "domain": "example.com",
   "records": ["AAVpc3N1ZWxldHNlbmNyeXB0Lm9yZw=="]
 }
 "error": "HTTP found unexpected value at LIS perspective. CAA check failed at the JFK perspective.",
}

## `dns` method

Required request fields

* `domain`
* `record-type`
* `prefix`
* `expected`

Optional

* `caa` Defaults to true.

Response is same as with `http`.


# Operation

TODO describe operation of each.

# Authentication

Describe usage of `Authorization` header.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
