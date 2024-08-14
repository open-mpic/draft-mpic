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
    fullname: "Bas Westerbaan"
    organization: Cloudflare
    email: "bas@cloudflare.com"

normative:

informative:


--- abstract

This memo defines an API for a service to offer
multi-perspective issuance corroboration (MPIC)
for domain control validation.

TODO CA/B context.


--- middle

# Introduction

TODO References to CA/B


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# API

A service is identified by a HTTPS url.
As a running example, say `https://mpc.example.com/staging`.

A client requests an MPIC from the service
by sending a POST request to the resource `/mpic/draft-00`
below the service URL.
In the running example `https://mpc.example.com/staging/mpic/draft-00`.

[[ The final version of the API will use `/mpic/v1`. Incompatible
   versions of the draft will bump the `-00`. ]]

The body of the POST is a JSON object that described the MPIC request.
The service will respond with a JSON object.
There are three different method, described below. The request object has
a `method` field that distinguishes the method.

## `caa` method

A `caa` requests asks the MPIC service to retrieve the relevant CAA DNS
records for a given domain from multiple vantage points.

The request object has the following specific fields.

* `domain` The domain to check the CAA records for.

If successful (described in TODO REF below), the response object
contains a `ok` field set to `true`, and
an `caa` field, which itself is an object with two fields:

* `domain` The domain on which the CAA records were found. This could be
  a parent domain of the requested `domain`.

* `records` A list of base64 encoded CAA records.

On failure, the response object will have the `ok` field set to `false`,
and an `error` field describing the error.

[[ TODO do we to define the possible errors, or at least assign
        some codes? ]]

An example request is given below

~~~
POST /staging/mpic/draft-00
Host: mpc.example.com
Content-Type: application/json

{
 "method": "caa",
 "domain": "some.example.com"
}
~~~

An example of a response for a succesful validation.

~~~
{
 "ok": true,
 "caa": {
  "domain": "example.com",
  "records": ["AAVpc3N1ZWxldHNlbmNyeXB0Lm9yZw=="]
 }
}
~~~

An example of a response for an unsuccesful validation.

~~~
{
 "ok": false,
 "error": "LIS sawrecord 'xyz' on example.com which was not present from vantage point LIS"
}
~~~

[[ TODO How much information to return on error to help debug, and how
   structured should it be? I'd say it's good to be helpful, but it's bad
   to be structured as it's less readable. ]]


## `http` method

TODO

Performs a GET from multiple vantage points, and checks whether the body
matches expectation.

Required request fields

* `domain`
* `path`
* `expected` Expected body [[ TODO what if it's not UTF-8? ]]

Makes request to `http://[domain][path]`.

Optional request fields

* `caa` Performs CAA validation at the same time for the domain as
  described above. Defaults to true.

Response object.

Contains `ok` and `error` as in `caa` method.

If `caa` was set to true, contains `caa` field as described above.

TODO example

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
