---
title: "Multi-Perspective Issuance Corroboration (MPIC) Service "
abbrev: "MPIC"
category: info

docname: draft-westerbaan-secdispatch-mpic-latest
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
  group: "Security Dispatch"
  type: "Working Group"
  mail: "secdispatch@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/secdispatch/"
  github: "bwesterb/draft-mpic"
  latest: "https://bwesterb.github.io/draft-mpic/draft-westerbaan-secdispatch-mpic.html"

author:
 -
    fullname: "Syed Suleman Ahmad"
    organization: Cloudflare
    email: "suleman@cloudflare.com"
 -
    fullname: "Bas Westerbaan"
    organization: Cloudflare
    email: "bas@cloudflare.com"
normative:
 - RFC8555

informative:
 - RFC6570


--- abstract

This memo defines an API for a service to offer multi-perspective issuance
corroboration (MPIC) for domain control validation. Ballot SC-67 v3 of CA/B
forum requires MPIC to be performed by all certification authorities (CAs) in
the Web PKI. This API allows CAs to use external MPIC infrastructure.

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
Several, but not all, CAs use the specific DCV methods of ACME {{RFC855}}.
Domain control validation is vulnerable to DNS and BGP hijacks.
These can be partially mitigated by performing DCV from multiple
vantage points, which is dubbed "multiple perspective issuance corroboration" (MPIC).
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

A MPIC service is identified by a HTTPS url.
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

## `caa` validation method

A `caa` requests asks the MPIC service to retrieve the relevant CAA DNS
records for a given domain from multiple vantage points.

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
 "error": "LIS saw record 'xyz' on example.com which was not present from vantage point LIS"
}
~~~

[[ TODO How much information to return on error to help debug, and how
   structured should it be? I'd say it's good to be helpful, but it's bad
   to be structured as it's less readable. ]]


## `http` method
A `http` requests the MPIC server to perform ACME HTTP challenge validation
{{RFC8555}} for the domain's HTTP server from each distributed vantage point.

Performs a GET from multiple vantage points, and checks whether the body matches
expectation. Optionally, it allows performing an additional CAA record lookup
for the domain.

The request JSON object has the following specific fields.

* `domain` (required, string): The domain name being verified.
* `path` (required, string): The path at which the ACME HTTP challenge resource is provisioned.
* `expected` (required, string): Expected body of the response [[ TODO what if it's not UTF-8? ]]
* `caa-check` (optional, boolean): Performs CAA validation at the same time for the domain as described above. Defaults to true.

~~~
POST /staging/mpic/draft-00
Host: mpc.example.com
Content-Type: application/json

{
 "method": "http",
 "domain": "some.example.com",
 "path": ".well-known/acme-challenge/token",
 "expected": "challenge_token",
 "caa-check": false,
}
~~~

The MPIC server constructs a URL by populating the URL template
{{RFC6570}}, `http://{domain}/{path}`, and verifies that the resulting URL is
well-formed, before making a HTTP GET request to the URL from each vantage
point. Each vantage point SHOULD follow redirects when dereferencing the URL.
The MPIC server verifies that `expected` value provided by the client matches
with the body of the response received from each vantage point.

If the above verifications succeeds, then the validation is successful. If
the request fails, or the body does not pass these checks, then it has failed.

Along side, the MPIC server queries for the CAA records for the
`domain` if the `caa-check` request parameter is set to "true".

On success, the response object contains a top-level `success` field set to
`true`, and `checks` field which itself is an object of two fields:

* `http-check` (required, object): Contains an indentifer for validation result.
    * `success` (required, boolean): Indicates the success of the HTTP challenge validation from each vantage point.

* `caa-check` (required, array of object): If `caa-check` is set to true, contains an array of identifier objects that the order pertains to. Otherwise, it is omitted.
    * `success` (required, boolean): Indicates the success of consistent CAA record response from each vantage point.
    * `caa` (required, object): Contains `caa` object field as described [TODO add ref] in the above subsection.

An example of a response for a succesful validation with `caa-check` set to
false.

~~~
{
 "success": true,
 "checks": {
  "http-check": {
    "success": true
  }
 }
}
~~~

An example of a response for a succesful validation with `caa-check` set to
true.

~~~
{
 "success": true,
 "checks": {
  "http-check": {
    "success": true
  },
  "caa-check": {
    "success": true,
    "caa": {
      "domain": "example.com",
      "records": ["AAVpc3N1ZWxldHNlbmNyeXB0Lm9yZw=="]
    }
  }
 }
}
~~~

On failure, the response object will have the top-level `success` field set to
`false`, and the `checks` field describing corresponding error details specific
to each validation method failure desscription. A separate top-level `error`
field describes the error.

* `error` (required, string): Error message.

An example of a response where the validation failed with caa-check set to true, and the `caa` method being successful.

~~~
{
 "success": false,
 "error": "HTTP method validation failed",
 "checks": {
   "http-check": {
    "success": false,
    "error": "HTTP ACME challenge validation failed at LIS"
   },
   "caa-check": {
     "success": true,
     "caa": {
      "domain": "example.com",
      "records": ["AAVpc3N1ZWxldHNlbmNyeXB0Lm9yZw=="]
    }
   }
 }
}
~~~

Similarly example of a response where caa-check set to true, and both
methods fail.

~~~
{
 "success": false,
 "error": "HTTP and CAA methods both failed",
 "checks": {
   "http-check": {
    "success": false,
    "error": "HTTP ACME challenge validation failed at LIS"
   },
   "caa-check": {
     "success": false,
     "error": "LIS saw record 'xyz' on example.com which was not present from vantage point LIS"
   }
 }
}
~~~


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
