---
layout: post
title: "Regional Analytics Consent for Static Sites with CloudFront and Terragrunt"
date: 2026-11-14
description: "Add country and US-state consent rules to an S3 static site with CloudFront headers, an edge endpoint, Consent Mode and Terragrunt. No paid geolocation database needed."
tags: [aws, cloudfront, terragrunt, devops, static-sites, privacy, google-analytics]
draft: false
---

A static site can be built by GitHub Actions, uploaded to S3 and served through
CloudFront without an application server. Adding Google Analytics is also easy:
paste a tag into the page template. Deciding **when that tag may load** takes
more care.

The decision can depend on a country, a US state and a visitor's saved choice.
The HTML is the same for everyone, but the analytics default should not always
be the same. I implemented this on five static sites, including
[duns.me](https://duns.me/) and [scomp.me](https://scomp.me/), using a small
CloudFront endpoint. The pages stay static and share the same cache. The edge
answers one dynamic question.

<!--more-->

There is no paid MaxMind subscription, browser call to an IP lookup provider,
or application server in this design. CloudFront already has connection
geolocation. A CloudFront Function turns that information into a policy, and
the browser applies the policy before loading Google Analytics or Google Tag
Manager.

## Decide the policy before writing the banner

Privacy rules do not form a single worldwide cookie switch. A rule about
targeted advertising is not necessarily a rule about ordinary site analytics.
A law can also have revenue thresholds, processing-volume thresholds,
sensitive-data restrictions and provisions that start on different dates.

| Situation | What the implementation needs to distinguish |
| --- | --- |
| EEA and UK | Non-essential tracking commonly requires prior consent; limited exemptions depend on the use and jurisdiction. |
| Other countries | Consent, notice, lawful-basis and transfer rules vary. A general privacy-law label does not prove that this particular analytics configuration needs a popup. |
| United States | Many state laws provide opt-out rights for sale or targeted advertising and additional sensitive-data rules. They do not impose a general nationwide analytics-cookie acceptance requirement. |
| Future legislation | An adopted law, a proposed postponement and an effective law are different policy inputs. |

For my November 14, 2026 rollout, I chose a **precautionary analytics policy**:
126 countries and jurisdictions, plus 20 US states, receive an opt-in prompt.
This expands the original EEA, UK and Switzerland list. The country expansion
uses the 125 jurisdictions marked `inforce` and `optin` in the November 10
Digital Privacy Regulations register, excluding its EU aggregate, and keeps
Switzerland. Macau and Taiwan have separate country codes in the list.

The selected US states are California, Colorado, Connecticut, Delaware,
Florida, Indiana, Iowa, Kentucky, Maryland, Minnesota, Montana, Nebraska,
New Hampshire, New Jersey, Oregon, Rhode Island, Tennessee, Texas, Utah and
Virginia. Requiring acceptance in those states is my policy choice, rather
than a claim that each state requires this exact popup.

The [consent map](https://digitalprivacyregs.com/consent-models) and
[calendar](https://digitalprivacyregs.com/calendar) are useful review leads.
They combine different processing purposes, and some status labels can lag
commencement. For example, Canada's regulator permits opt-out consent for
online behavioural advertising under specified conditions. Verify the rules
for your processing instead of importing a marketing classification as a
legal conclusion. A banner also does not resolve data localization or
international-transfer obligations by itself.

Keep a dated list in infrastructure code. Record sources, review upcoming
effective dates and change the list deliberately. A country omitted from an
opt-in list is not thereby certified exempt from privacy law.

## Let the edge answer a small question

The request flow is:

```text
GET /article/       -> CloudFront -> cached static HTML from S3
                                  -> browser starts with analytics denied

GET /privacy-region -> CloudFront Function -> uncached policy JSON
                       no S3 request and no application server

browser             -> check saved choice and browser privacy signal
                    -> show consent, or enable the permitted regional default
                    -> load the Google tag only when analytics is enabled
```

The endpoint returns a policy, not the visitor's IP address, country or state:

```json
{"policy":"opt-in","version":1}
```

The other successful result is `{"policy":"no-prompt","version":1}`.
`no-prompt` means the site's regional default permits analytics. It is **not**
an assertion that the visitor accepted anything.

An explicit rejection overrides location. In this implementation, Global
Privacy Control also disables analytics everywhere, even after an earlier
acceptance. A failed lookup, an unsupported response version or missing US
state information leads to opt-in.

## Why forwarding a header to the page is not enough

CloudFront can add a country header to the request it forwards to S3. S3
still returns the object stored in the bucket. It does not turn
`CloudFront-Viewer-Country` into a variable inside `index.html`.

Adding a response header is a different operation, but JavaScript running in
the document cannot directly inspect the headers of that document's original
navigation response. Those headers are visible in developer tools; they are
not a JavaScript variable supplied to the page.

An additional `fetch` can read response headers. You could make a viewer
response function add a policy header and issue `HEAD /` from the browser.
That is a valid alternative if the header is calculated for each visitor and
browser/intermediary caching cannot reuse someone else's policy. It still
requires another request and careful cache handling. A dedicated JSON route
provides a clearer contract without coupling privacy decisions to the home
page's redirects, caching or content.

There are other valid designs:

| Approach | Operational consequence |
| --- | --- |
| Browser calls a third-party IP service | Another dependency receives the lookup request; consider its privacy terms, limits, latency and failure behaviour. |
| Your backend queries a geolocation database | You operate an API, database updates, licensing and availability. GeoLite has free licensing options; a paid MaxMind product is not inherently required. |
| Server rendering or HTML personalization at the edge | The server can embed a policy, but the HTML and cache strategy must account for geography. CloudFront Functions cannot edit an existing response body. |
| Browser geolocation API | It requires a permission request for coordinates, which is unnecessary for this task. |
| Ask everyone for consent | Simple and often reasonable, but it gives up the regional behaviour. |

The endpoint is therefore not the only possible solution. It is a good fit
when the constraints are **static HTML, S3, CloudFront and no additional
geolocation service or application backend**.

## CloudFront headers already provide the location

CloudFront derives location from the connection IP address. Its geographic
headers include:

| Header | Value and possible use |
| --- | --- |
| `CloudFront-Viewer-Country` | Two-letter ISO 3166-1 country code, such as `DE` or `US`. |
| `CloudFront-Viewer-Country-Region` | First-level subdivision code, up to three characters; for a US connection, a state code such as `CA` or `NY`. |
| `CloudFront-Viewer-Country-Name` | Human-readable country name. |
| `CloudFront-Viewer-Country-Region-Name` | Human-readable subdivision name. |
| `CloudFront-Viewer-City` | City, where geolocation is sufficiently specific. |
| `CloudFront-Viewer-Postal-Code` | Postal code, where available. |
| `CloudFront-Viewer-Time-Zone` | IANA time zone, such as `America/New_York`. |
| `CloudFront-Viewer-Latitude`, `CloudFront-Viewer-Longitude` | Approximate geographic coordinates. |
| `CloudFront-Viewer-Metro-Code` | US metro-area code. |

The consent endpoint needs only the first two. Forwarding coordinates, city
and postal code adds detail without helping this decision.

Enable the selected headers on the relevant behaviour through an **origin
request policy**. AWS's CloudFront Functions geolocation example documents
that the headers must be enabled by an origin request policy or cache policy
to appear in the Function request. For this design, use the origin request
policy and keep geography out of the static cache key.

The extended location headers, including country-region, are not added for
requests originating from the AWS network. Some locations cannot be resolved
precisely enough either. Missing state data must be an expected input.

IP geolocation estimates a connection's location, not legal residency. VPNs,
proxies and datacenter egress can change the result. The header is useful for
a default, while saved choices, GPC and always-available privacy controls
provide independent ways to decline tracking.

## Keep countries and states in one shared policy

Here is the complete rollout snapshot, as an example `consent.hcl`. It is a
dated precautionary configuration, not a list of legally mandatory analytics
banners. Review it before adopting it and keep future-law review notes with
your infrastructure documentation.

```hcl
# Precautionary policy snapshot, reviewed 2026-11-14.
# Review applicability and future changes before adopting these lists.
locals {
  analytics_opt_in_countries = [
    "AD", "AL", "AM", "AO", "AR", "AT", "AZ", "BA", "BB", "BE", "BF", "BG",
    "BH", "BJ", "BN", "BR", "BS", "BW", "BY", "BZ", "CA", "CD", "CG", "CH",
    "CI", "CN", "CO", "CR", "CU", "CV", "CY", "CZ", "DE", "DK", "DO", "DZ",
    "EC", "EE", "EG", "ES", "ET", "FI", "FR", "GA", "GB", "GE", "GH", "GN",
    "GQ", "GR", "HR", "HU", "ID", "IE", "IL", "IS", "IT", "JM", "JO", "KE",
    "KG", "KR", "KZ", "LB", "LC", "LI", "LT", "LU", "LV", "MA", "MC", "MD",
    "ME", "MK", "ML", "MN", "MO", "MR", "MT", "MU", "MW", "MY", "NE", "NG",
    "NL", "NO", "NP", "OM", "PA", "PE", "PH", "PL", "PT", "QA", "RO", "RS",
    "RU", "RW", "SA", "SC", "SE", "SI", "SK", "SM", "SN", "SO", "ST", "SV",
    "SZ", "TD", "TG", "TH", "TJ", "TM", "TN", "TR", "TW", "TZ", "UA", "UG",
    "UY", "UZ", "VN", "ZA", "ZM", "ZW",
  ]

  # Comprehensive state laws in effect per the reviewed tracker; opt-in is
  # our precautionary choice, rather than a blanket US cookie requirement.
  analytics_opt_in_us_states = [
    "CA", "CO", "CT", "DE", "FL", "IA", "IN", "KY", "MD", "MN",
    "MT", "NE", "NH", "NJ", "OR", "RI", "TN", "TX", "UT", "VA",
  ]
}
```

The edge checks state codes only when the country is `US`. `CA` as a country
means Canada; `CA` as a US subdivision means California. Mixing those two
namespaces is an easy configuration error.

## CloudFront Function and Terraform module

A CloudFront Function is enough for a small request-time decision. It is not a
Lambda function: it uses the CloudFront Functions runtime and event format.
It also cannot make an HTTP call to an external geolocation API. That is not
needed here because CloudFront supplies the headers.

For reuse across repositories, we publish an MIT-licensed
[CloudFront regional consent module](https://github.com/BOPOHA/terraform-aws-cloudfront-regional-consent).
The first release is [v0.1.0](https://github.com/BOPOHA/terraform-aws-cloudfront-regional-consent/releases/tag/v0.1.0),
with Terraform validation, mocked-provider tests and tests that execute the
generated JavaScript in CI. A local module is useful while developing the
design; a public release gives independent sites a documented interface and
a version they can upgrade deliberately. Policy lists still belong to each
caller rather than being hardcoded into the reusable module.

The following is a complete small local module, `modules/consent-edge/main.tf`,
included here so the article remains self-contained.
The AWS provider configuration comes from the caller. Terraform inserts the
two lists into the JavaScript using `jsonencode`, so policy data stays outside
the handwritten handler logic.

```hcl
variable "name" {
  type = string
}

variable "analytics_opt_in_countries" {
  type = list(string)
  validation {
    condition     = alltrue([for code in var.analytics_opt_in_countries : can(regex("^[A-Z]{2}$", code))])
    error_message = "Use uppercase two-letter country codes."
  }
}

variable "analytics_opt_in_us_states" {
  type = list(string)
  validation {
    condition     = alltrue([for code in var.analytics_opt_in_us_states : contains(split(",", "AL,AK,AZ,AR,CA,CO,CT,DE,FL,GA,HI,ID,IL,IN,IA,KS,KY,LA,ME,MD,MA,MI,MN,MS,MO,MT,NE,NV,NH,NJ,NM,NY,NC,ND,OH,OK,OR,PA,RI,SC,SD,TN,TX,UT,VT,VA,WA,WV,WI,WY,DC"), code)])
    error_message = "Use valid uppercase US state or DC codes."
  }
}

resource "aws_cloudfront_function" "consent" {
  name    = var.name
  runtime = "cloudfront-js-2.0"
  publish = true
  code    = <<-JAVASCRIPT
    var OPT_IN_COUNTRIES = ${jsonencode(var.analytics_opt_in_countries)};
    var OPT_IN_US_STATES = ${jsonencode(var.analytics_opt_in_us_states)};

    function privacyRegion(request, optInCountries, optInUSStates) {
      var allowed = request.method === 'GET' || request.method === 'HEAD';
      var header = request.headers['cloudfront-viewer-country'];
      var country = header ? header.value : '';
      var known = /^[A-Z]{2}$/.test(country) && country !== 'XX';
      var policy = !known || optInCountries.indexOf(country) !== -1 ? 'opt-in' : 'no-prompt';
      if (country === 'US' && policy !== 'opt-in') {
        var regionHeader = request.headers['cloudfront-viewer-country-region'];
        var region = regionHeader ? regionHeader.value : '';
        var knownUSRegions = 'AL,AK,AZ,AR,CA,CO,CT,DE,FL,GA,HI,ID,IL,IN,IA,KS,KY,LA,ME,MD,MA,MI,MN,MS,MO,MT,NE,NV,NH,NJ,NM,NY,NC,ND,OH,OK,OR,PA,RI,SC,SD,TN,TX,UT,VT,VA,WA,WV,WI,WY,DC'.split(',');
        if (knownUSRegions.indexOf(region) === -1 || (optInUSStates || []).indexOf(region) !== -1) policy = 'opt-in';
      }
      var headers = {
        'content-type': { value: 'application/json; charset=utf-8' },
        'cache-control': { value: 'private, no-store' },
        'x-content-type-options': { value: 'nosniff' }
      };
      if (!allowed) headers.allow = { value: 'GET, HEAD' };
      return {
        statusCode: allowed ? 200 : 405,
        statusDescription: allowed ? 'OK' : 'Method Not Allowed',
        headers: headers,
        body: request.method === 'HEAD' ? '' : JSON.stringify(allowed ? { policy: policy, version: 1 } : { error: 'method not allowed' })
      };
    }

    function handler(event) {
      var request = event.request;
      if (request.uri === '/privacy-region') {
        return privacyRegion(request, OPT_IN_COUNTRIES, OPT_IN_US_STATES);
      }
      // Merge existing redirects and other routing rules here.
      if (request.uri.endsWith('/')) request.uri += 'index.html';
      return request;
    }
  JAVASCRIPT
}

resource "aws_cloudfront_origin_request_policy" "consent" {
  name = "${var.name}-geolocation"
  cookies_config {
    cookie_behavior = "none"
  }
  headers_config {
    header_behavior = "whitelist"
    headers {
      items = ["CloudFront-Viewer-Country", "CloudFront-Viewer-Country-Region"]
    }
  }
  query_strings_config {
    query_string_behavior = "none"
  }
}

output "function_arn" {
  value = aws_cloudfront_function.consent.arn
}

output "origin_request_policy_id" {
  value = aws_cloudfront_origin_request_policy.consent.id
}
```

The Function handles `GET` and `HEAD`, rejects other methods with `405`, and
sets `Cache-Control: private, no-store`. A response generated at the viewer
request stage does not enter CloudFront's object cache or reach S3. Browser
and intermediary caching must also be disabled, because this is a per-visitor
answer.

The country/state headers are forwarded without joining the cache key. The
normal HTML and asset objects can remain cached and shared across locations.
In this example, requests reaching S3 can carry the two headers; S3 does not
use them to render content.

The final few lines preserve a simple `/path/` to `/path/index.html` rewrite.
Real sites may also have canonical redirects, federation endpoints, downloads
and custom error handling. Merge the privacy branch into the existing
viewer-request handler. You cannot stack two independent CloudFront Functions
at the same event on one behaviour. If you have no existing rewrite function,
a dedicated `/privacy-region` behaviour can limit the new Function to that
route instead of invoking it for every page and asset.

In `live/prod/consent-edge/terragrunt.hcl`, read the shared policy and pass it
to the module:

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "${get_repo_root()}//modules/consent-edge"
}

locals {
  consent = read_terragrunt_config("${get_repo_root()}/consent.hcl")
}

inputs = {
  name                       = "example-com-consent"
  analytics_opt_in_countries = local.consent.locals.analytics_opt_in_countries
  analytics_opt_in_us_states = local.consent.locals.analytics_opt_in_us_states
}
```

### Use the public release instead

For the public module, keep the same include and policy locals, and replace
the source and inputs with:

```hcl
terraform {
  # v0.1.0, pinned to its immutable release commit.
  source = "git::https://github.com/BOPOHA/terraform-aws-cloudfront-regional-consent.git?ref=f025f4c6ebd77ee7adca259f738516656f6e8394"
}

inputs = {
  name                       = "example-com-consent"
  privacy_region_enabled     = true
  analytics_opt_in_countries = local.consent.locals.analytics_opt_in_countries
  analytics_opt_in_us_states = local.consent.locals.analytics_opt_in_us_states
}
```

The public module keeps the resource and output names of our existing routing
module to allow a source migration without recreating resources. Its Function
output is `arn`, and its policy output is `privacy_origin_request_policy_id`.
In the CDN examples below, replace
`dependency.consent.outputs.function_arn` with `dependency.consent.outputs.arn`,
and `dependency.consent.outputs.origin_request_policy_id` with
`dependency.consent.outputs.privacy_origin_request_policy_id` when using the
public release. The local teaching module above uses the shorter example
output names.

The endpoint is disabled by default in the public module. Enabling it is
explicit, and the caller must provide its reviewed country and US-state lists.
Optional canonical redirects and SEP-2 federation support are also included;
merge any other existing request logic carefully before changing a production
association. Preserve the current provider lock, run a plan, and check resource
addresses and code changes before applying. A GitHub repository and release
can be used directly; Terraform Registry listing is a separate publication
step and is not required by this Git source.

## Attach the policy to a community CDN module

Creating the policy is insufficient. The behaviour executing the Function
must reference its `origin_request_policy_id`, and the Function itself must
be associated with that behaviour.

The two module interfaces below are different. These are **CDN configuration
excerpts**: retain the existing origins, TLS certificate, aliases, DNS, S3
access control and ordered behaviours from your working configuration. The
module versions shown are the versions checked for these examples, not a
recommendation to upgrade or downgrade a working distribution.

For `cloudposse/cloudfront-s3-cdn/aws` version `2.1.1`, the default-behaviour
policy input is at the module's top level. Function associations are a list:

```hcl
terraform {
  source = "tfr:///cloudposse/cloudfront-s3-cdn/aws?version=2.1.1"
}

dependency "consent" {
  config_path = "../consent-edge"
}

inputs = {
  # Keep the remaining inputs from your existing CDN configuration.
  cache_policy_id          = "658327ea-f89d-4fab-a63d-7e88639e58f6"
  origin_request_policy_id = dependency.consent.outputs.origin_request_policy_id

  function_association = [
    {
      event_type   = "viewer-request"
      function_arn = dependency.consent.outputs.function_arn
    }
  ]
}
```

For `terraform-aws-modules/cloudfront/aws` version `5.1.0`, the policy belongs
inside the cache behaviour. Its function associations use a map keyed by
event type:

```hcl
terraform {
  source = "tfr:///terraform-aws-modules/cloudfront/aws?version=5.1.0"
}

dependency "consent" {
  config_path = "../consent-edge"
}

inputs = {
  # Retain origins, TLS, OAC, aliases and other existing inputs.
  default_cache_behavior = {
    target_origin_id       = "site-s3" # Use your existing origin ID.
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    compress               = true

    cache_policy_id          = "658327ea-f89d-4fab-a63d-7e88639e58f6"
    origin_request_policy_id = dependency.consent.outputs.origin_request_policy_id

    function_association = {
      viewer-request = {
        function_arn = dependency.consent.outputs.function_arn
      }
    }
  }
}
```

`658327ea-f89d-4fab-a63d-7e88639e58f6` is AWS's managed `CachingOptimized`
cache policy. It is suitable for the static objects in this example. Preserve
your current policy if the site has different caching requirements.

Each behaviour has one origin request policy. If yours already forwards
headers, cookies or query strings needed by the origin, merge the geographic
headers into that policy instead of blindly replacing it with the minimal
one above. Review the plan for unintended routing and access changes.

## Lambda@Edge alternative

If the site already uses Lambda@Edge, or its edge logic needs capabilities
outside CloudFront Functions, Lambda@Edge can answer the same endpoint. Its
event shape is different: headers are arrays, the request is under
`Records[0].cf.request`, and response status is a string.

Use an **origin-request** trigger for this example. CloudFront-added
geographic headers are available there when configured by a cache or origin
request policy. Lambda@Edge viewer-request triggers run before those headers
are added. Do not copy the Function example into a viewer-request Lambda and
assume it receives identical inputs.

The following is a complete `index.js`, with the dated example policy
embedded. In production, generate that embedded data from the same shared
policy instead of maintaining a separate handwritten list. No external lookup
is performed.

```javascript
// Handler: index.handler. Dated precautionary policy; review before deploying.
const policy = {
  countries: [
    "AD", "AL", "AM", "AO", "AR", "AT", "AZ", "BA", "BB", "BE", "BF", "BG",
    "BH", "BJ", "BN", "BR", "BS", "BW", "BY", "BZ", "CA", "CD", "CG", "CH",
    "CI", "CN", "CO", "CR", "CU", "CV", "CY", "CZ", "DE", "DK", "DO", "DZ",
    "EC", "EE", "EG", "ES", "ET", "FI", "FR", "GA", "GB", "GE", "GH", "GN",
    "GQ", "GR", "HR", "HU", "ID", "IE", "IL", "IS", "IT", "JM", "JO", "KE",
    "KG", "KR", "KZ", "LB", "LC", "LI", "LT", "LU", "LV", "MA", "MC", "MD",
    "ME", "MK", "ML", "MN", "MO", "MR", "MT", "MU", "MW", "MY", "NE", "NG",
    "NL", "NO", "NP", "OM", "PA", "PE", "PH", "PL", "PT", "QA", "RO", "RS",
    "RU", "RW", "SA", "SC", "SE", "SI", "SK", "SM", "SN", "SO", "ST", "SV",
    "SZ", "TD", "TG", "TH", "TJ", "TM", "TN", "TR", "TW", "TZ", "UA", "UG",
    "UY", "UZ", "VN", "ZA", "ZM", "ZW",
  ],
  us_states: [
    "CA", "CO", "CT", "DE", "FL", "IA", "IN", "KY", "MD", "MN", "MT", "NE",
    "NH", "NJ", "OR", "RI", "TN", "TX", "UT", "VA",
  ]
};

function privacyRegion(request, optInCountries, optInUSStates) {
  var allowed = request.method === 'GET' || request.method === 'HEAD';
  var header = request.headers['cloudfront-viewer-country'];
  var country = header ? header.value : '';
  var known = /^[A-Z]{2}$/.test(country) && country !== 'XX';
  var policy = !known || optInCountries.indexOf(country) !== -1 ? 'opt-in' : 'no-prompt';
  if (country === 'US' && policy !== 'opt-in') {
    var regionHeader = request.headers['cloudfront-viewer-country-region'];
    var region = regionHeader ? regionHeader.value : '';
    var knownUSRegions = 'AL,AK,AZ,AR,CA,CO,CT,DE,FL,GA,HI,ID,IL,IN,IA,KS,KY,LA,ME,MD,MA,MI,MN,MS,MO,MT,NE,NV,NH,NJ,NM,NY,NC,ND,OH,OK,OR,PA,RI,SC,SD,TN,TX,UT,VT,VA,WA,WV,WI,WY,DC'.split(',');
    if (knownUSRegions.indexOf(region) === -1 || (optInUSStates || []).indexOf(region) !== -1) policy = 'opt-in';
  }
  var headers = {
    'content-type': { value: 'application/json; charset=utf-8' },
    'cache-control': { value: 'private, no-store' },
    'x-content-type-options': { value: 'nosniff' }
  };
  if (!allowed) headers.allow = { value: 'GET, HEAD' };
  return {
    statusCode: allowed ? 200 : 405,
    statusDescription: allowed ? 'OK' : 'Method Not Allowed',
    headers: headers,
    body: request.method === 'HEAD' ? '' : JSON.stringify(allowed ? { policy: policy, version: 1 } : { error: 'method not allowed' })
  };
}

exports.handler = (event, context, callback) => {
  const request = event.Records[0].cf.request;
  if (request.uri !== '/privacy-region') return callback(null, request);
  const headers = {};
  for (const name of ['cloudfront-viewer-country', 'cloudfront-viewer-country-region']) {
    const value = request.headers[name];
    if (value && value[0]) headers[name] = { value: value[0].value };
  }
  const reply = privacyRegion({ method: request.method, headers }, policy.countries, policy.us_states);
  const outputHeaders = {};
  for (const name of Object.keys(reply.headers)) {
    outputHeaders[name] = [{ key: name, value: reply.headers[name].value }];
  }
  callback(null, {
    status: String(reply.statusCode),
    statusDescription: reply.statusDescription,
    headers: outputHeaders,
    bodyEncoding: 'text',
    body: reply.body
  });
};
```

For example, generate the embedded policy object from the same Terraform
variables:

```hcl
locals {
  lambda_policy_json = jsonencode({
    countries = var.analytics_opt_in_countries
    us_states = var.analytics_opt_in_us_states
  })
}

# In your packaging step, embed local.lambda_policy_json as the policy object
# in index.js. Configure the Lambda handler as index.handler.
```

An origin-request Lambda runs on a cache miss. Therefore put `/privacy-region`
on a **dedicated behaviour with caching disabled**; otherwise a cached answer
could prevent the Lambda from making a fresh decision. An ordered-behaviour
entry for the terraform-aws-modules interface looks like this:

```hcl
{
  path_pattern           = "/privacy-region"
  target_origin_id       = "site-s3"
  viewer_protocol_policy = "redirect-to-https"
  allowed_methods        = ["GET", "HEAD", "OPTIONS"]
  cached_methods         = ["GET", "HEAD"]

  # AWS managed CachingDisabled; default static behaviour stays cached.
  cache_policy_id          = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"
  origin_request_policy_id = dependency.consent.outputs.origin_request_policy_id

  lambda_function_association = {
    origin-request = {
      lambda_arn   = dependency.lambda.outputs.qualified_arn
      include_body = false
    }
  }
}
```

This is an entry to merge into `ordered_cache_behavior`, not a complete CDN
configuration. Supply the Lambda dependency from your own packaging stack.
Do not also attach the endpoint-producing CloudFront Function to this
behaviour: its early response would make the Lambda path unnecessary.

Lambda@Edge requires a function in `us-east-1`, an associated **numbered
version**, a supported runtime, and an execution role trusted by both
`lambda.amazonaws.com` and `edgelambda.amazonaws.com`. `$LATEST` and aliases
are not valid associations. Custom environment variables are unavailable,
which is another reason to embed policy data in the deployment package. Lambda@Edge also
has a different pricing and deployment model. For a small synchronous policy
lookup, the CloudFront Function is the simpler choice.

## The browser still needs real consent controls

AWS answers the regional question. It does not collect consent, set Google
Consent Mode, or stop a tag already embedded elsewhere in the page.

Remove the old direct `gtag.js` or GTM loader, inline initializers and any GTM
`noscript` tracking iframe. Initialize Consent Mode before loading the tag.
Keep advertising permissions denied unless you have designed and reviewed a
separate advertising-consent flow.

For a plain static page, put controls like these in the document. The buttons
should be equally easy to find and use; access to the page remains available
without analytics.

```html
<style>
  [hidden] { display: none !important; }
</style>

<section hidden data-regional-consent
  data-storage-key="example-analytics-consent-v1"
  data-google-id="G-XXXXXXXXXX"
  data-google-mode="ga4"
  aria-labelledby="analytics-consent-title">
  <h2 id="analytics-consent-title">Optional analytics</h2>
  <p>Allow Google Analytics to measure visits and site usage?
    <a href="/privacy/">Read the privacy policy</a>.</p>
  <button type="button" data-consent="denied">Essential only</button>
  <button type="button" data-consent="granted">Allow analytics</button>
</section>

<button type="button" data-privacy-settings>Privacy settings</button>
```

Replace the example measurement ID. For an analytics-only GTM container, use
`data-google-mode="gtm"` and your `GTM-...` ID. The data attributes supply
configuration; they are not Google script loaders.

Load the following controller after the controls exist, either inline at the
end of the document or through your site's own JavaScript bundle. It does not
need a JavaScript framework. The storage probe adds a conservative fallback:
if preferences cannot be stored, it does not automatically enable analytics
or reuse an old grant that the visitor may be unable to replace.

```javascript
// Configuration comes from the banner; geographic rules stay at the edge.
(() => {
  const banner = document.querySelector('[data-regional-consent]');
  if (!banner) return;
  const { storageKey, googleId, googleMode } = banner.dataset;
  const accept = banner.querySelector('[data-consent="granted"]');
  const gpc = navigator.globalPrivacyControl === true;
  const lifetime = 180 * 24 * 60 * 60 * 1000;
  let storageAvailable = false;
  try {
    const probeKey = `${storageKey}-availability`;
    localStorage.setItem(probeKey, '1');
    localStorage.removeItem(probeKey);
    storageAvailable = true;
  } catch { /* No automatic analytics when preferences cannot be saved. */ }
  let loaded = false;
  let revision = 0;

  window.dataLayer = window.dataLayer || [];
  function gtag() { window.dataLayer.push(arguments); }
  window.gtag = gtag;
  gtag('consent', 'default', {
    analytics_storage: 'denied', ad_storage: 'denied',
    ad_user_data: 'denied', ad_personalization: 'denied',
    functionality_storage: 'granted', security_storage: 'granted'
  });
  gtag('set', 'ads_data_redaction', true);
  gtag('set', 'url_passthrough', false);
  gtag('set', 'allow_google_signals', false);
  gtag('set', 'allow_ad_personalization_signals', false);
  gtag('set', 'allow_user_provided_data', false);

  function readChoice() {
    try {
      const raw = localStorage.getItem(storageKey);
      if (raw === 'granted' || raw === 'denied') return raw;
      const saved = JSON.parse(raw);
      // Optional migration for an earlier explicit analytics preference.
      if (saved?.version === undefined && typeof saved?.analytics === 'boolean') {
        return saved.analytics ? 'granted' : 'denied';
      }
      if (saved?.version !== 1 || !['granted', 'denied'].includes(saved.choice)) return null;
      if (!Number.isFinite(saved.updatedAt) || saved.updatedAt > Date.now() || Date.now() - saved.updatedAt > lifetime) return null;
      return saved.choice;
    } catch { return null; }
  }

  function showSettings() {
    banner.hidden = false;
    banner.classList.remove('hidden');
  }

  function clearAnalyticsCookies() {
    for (const part of document.cookie.split(';')) {
      const name = part.split('=')[0].trim();
      if (!/^(_ga|_gid|_gat)/.test(name)) continue;
      for (const domain of ['', `; domain=${location.hostname}`, `; domain=.${location.hostname}`]) {
        document.cookie = `${name}=; Max-Age=0; path=/${domain}; SameSite=Lax`;
      }
    }
  }

  function applyChoice(choice) {
    const granted = choice === 'granted' && !gpc;
    gtag('consent', 'update', { analytics_storage: granted ? 'granted' : 'denied' });
    if (granted && !loaded) {
      loaded = true;
      const script = document.createElement('script');
      script.async = true;
      if (googleMode === 'gtm') {
        window.dataLayer.push({ 'gtm.start': Date.now(), event: 'gtm.js' });
        script.src = `https://www.googletagmanager.com/gtm.js?id=${encodeURIComponent(googleId)}`;
      } else {
        gtag('js', new Date());
        gtag('config', googleId, {
          allow_google_signals: false, allow_ad_personalization_signals: false,
          allow_user_provided_data: false
        });
        script.src = `https://www.googletagmanager.com/gtag/js?id=${encodeURIComponent(googleId)}`;
      }
      document.head.append(script);
    } else if (!granted) {
      clearAnalyticsCookies();
      if (loaded) location.reload();
    }
    banner.hidden = true;
  }

  banner.addEventListener('click', event => {
    const button = event.target.closest('[data-consent]');
    if (!button || !['granted', 'denied'].includes(button.dataset.consent)) return;
    revision += 1;
    const choice = gpc ? 'denied' : button.dataset.consent;
    try {
      localStorage.setItem(storageKey, JSON.stringify({ version: 1, choice, updatedAt: Date.now() }));
    } catch { /* Choice still applies to this page when storage is unavailable. */ }
    applyChoice(choice);
  });
  for (const control of document.querySelectorAll('[data-privacy-settings]')) {
    control.addEventListener('click', event => {
      event.preventDefault();
      revision += 1;
      showSettings();
    });
  }
  if (gpc) {
    accept.disabled = true;
    accept.title = 'Analytics disabled by your browser privacy signal';
  }

  async function initialize() {
    const saved = storageAvailable ? readChoice() : null;
    if (gpc || saved) { applyChoice(gpc ? 'denied' : saved); return; }
    if (!storageAvailable) { showSettings(); return; }
    const initialRevision = revision;
    let policy = 'opt-in';
    try {
      const response = await fetch('/privacy-region', {
        cache: 'no-store', credentials: 'omit', signal: AbortSignal.timeout(3000)
      });
      if (!response.ok) throw new Error('Region lookup failed');
      const result = await response.json();
      if (result.version === 1 && result.policy === 'no-prompt') policy = 'no-prompt';
    } catch { /* Unknown location or a failed lookup requires explicit opt-in. */ }
    if (revision !== initialRevision) return;
    if (policy === 'no-prompt') applyChoice('granted');
    else showSettings();
  }
  initialize();
})();
```

New explicit choices expire after 180 days in this example. Automatic
analytics enabling is never persisted as acceptance. Legacy preference
formats are handled separately; keep only migrations your site needs. A
revision counter prevents a slow region response from overriding a visitor
who has already opened settings or chosen an option.

On withdrawal, the controller clears accessible `_ga`, `_gid` and `_gat`
cookies and reloads. Reloading matters because an already-loaded Google tag
cannot simply be removed from memory, and changing Consent Mode alone can
still permit cookieless requests. Check cookie names, paths and domains for
your own configuration. The example does not delete historical data held by
Google or manage unrelated vendors.

Keep the GTM container limited to the intended analytics tags and review any
new tags. Consent Mode signals are not a sandbox for custom HTML or vendors
that ignore them. Fonts, embeds and other third-party requests need their own
assessment; an analytics switch does not control every network request.

## Deployment and validation

Deploy the edge before the browser controller. If the endpoint is unavailable,
the controller requests consent rather than silently enabling analytics.
The existing GitHub Actions build and S3 upload can remain the publication
path; no service process needs to be deployed alongside the site.

For the example repository layout:

```bash
terragrunt plan --working-dir live/prod/consent-edge
terragrunt apply --working-dir live/prod/consent-edge

# Review the existing distribution before applying its policy association.
terragrunt plan --working-dir live/prod/cdn
terragrunt apply --working-dir live/prod/cdn

curl -i https://example.com/privacy-region
curl -I https://example.com/privacy-region
```

Expect JSON, a supported policy/version and `Cache-Control: private, no-store`.
`HEAD` has no body. Allowed-method configuration can reject a method before
the Function sees it; the handler's `405` is not a promise that every method
will reach it on every distribution.

Test the **rendered JavaScript**, not only the template, with synthetic
CloudFront events. For the Function API, a US-state event looks like this:

```json
{
  "version": "1.0",
  "context": {"eventType": "viewer-request"},
  "viewer": {"ip": "203.0.113.1"},
  "request": {
    "method": "GET",
    "uri": "/privacy-region",
    "querystring": {},
    "cookies": {},
    "headers": {
      "host": {"value": "example.com"},
      "cloudfront-viewer-country": {"value": "US"},
      "cloudfront-viewer-country-region": {"value": "CA"}
    }
  }
}
```

After saving it as `event.json`, test the deployed Function:

```bash
function_name=example-com-consent
etag=$(aws cloudfront describe-function --name "$function_name" \
  --stage LIVE --query ETag --output text)

aws cloudfront test-function --name "$function_name" --stage LIVE \
  --if-match "$etag" --event-object fileb://event.json
```

Include these boundaries in the tests:

| Input | Expected result for the policy above |
| --- | --- |
| `DE`, `ME`, `BR`, country `CA` | `opt-in` |
| `US` + state `CA` | `opt-in` |
| `US` + state `NY` | `no-prompt` |
| `US` + absent, lowercase or unrecognized state | `opt-in` |
| Missing country or `XX` | `opt-in` |
| `JP` + a stray state value `CA` | `no-prompt`; ignore US-state rules outside the US. |

Synthetic events prove the policy logic, but they do not prove that production
requests contain the configured headers. Use real HTTP requests from affected
and unaffected locations as well. Adding a country header to your local
`curl` request is not a substitute: CloudFront supplies its own geographic
metadata.

In my rollout, all five sites' rendered handlers and AWS LIVE Function tests
passed. Real HTTP probes returned opt-in from California and Brazil, and
no-prompt from New York and Japan. Residential New York probes also confirmed
the state-specific path on the four additional sites. Some datacenter probes
labelled New York returned opt-in, which is consistent with geolocation
differences or the missing-state fallback; probe labels are not proof of the
values CloudFront sees.

In a fresh browser profile, verify that no analytics script loads before
acceptance in an opt-in region. Also check automatic analytics in a no-prompt
region, saved rejection, GPC, a lookup timeout, blocked storage and withdrawal.
Keep the privacy-settings control usable in every region. Verify the existing
canonical redirects, downloads and real 404 responses after changing edge
logic. A geographically correct consent endpoint is not a successful rollout
if the site's routing or deployment data was damaged.

## Cost and operating the policy

CloudFront's built-in geolocation removes the need to buy, ship and update a
MaxMind database for this design. It does not make AWS usage free: account
for CloudFront requests and transfer, Function invocations, and any logging.
A viewer-request Function attached to the default behaviour runs for all
matching requests, including assets. Lambda@Edge has its own charges if you
choose that alternative.

The remaining operational responsibility is the policy itself. Review new
laws and effective dates, update the source list and matching privacy
disclosures, then deploy through the same infrastructure and site pipelines.
Record decisions as decisions rather than disguising a conservative popup
policy as a legal determination.

The useful capability here is small: AWS supplies request metadata, the edge
converts it into an uncached answer, and static browser code applies that
answer. The HTML remains an S3 object. The site gains the dynamic behaviour
it needs without becoming an application server.

## References

Implementation and policy review date: **November 14, 2026**.

1. [AWS: Add CloudFront request headers](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/adding-cloudfront-headers.html) — geographic fields, availability and policy configuration.
2. [AWS CloudFront Functions example: country-based redirects](https://github.com/aws-samples/amazon-cloudfront-functions/tree/main/redirect-based-on-country) — enabling geographic headers for the Function request.
3. [AWS: Origin request policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-origin-requests.html) — forwarding metadata independently of cache keys.
4. [AWS: Generate HTTP responses in request triggers](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-generating-http-responses-in-requests.html) — Lambda@Edge response generation and caching stages.
5. [AWS: Lambda@Edge restrictions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-at-edge-function-restrictions.html) — geographic-header timing, versions, region and runtime constraints.
6. [Google: Basic and advanced Consent Mode](https://developers.google.com/tag-platform/security/concepts/consent-mode) — the difference between delaying tag loading and sending signals from an already-loaded tag.
7. [Cloud Posse CloudFront S3 CDN module, version 2.1.1](https://registry.terraform.io/modules/cloudposse/cloudfront-s3-cdn/aws/2.1.1) — top-level policy input and list-shaped function associations.
8. [terraform-aws-modules CloudFront, version 5.1.0](https://github.com/terraform-aws-modules/terraform-aws-cloudfront/tree/v5.1.0) — behaviour-level policy inputs and event-keyed associations.
9. [Digital Privacy Regulations: consent models](https://digitalprivacyregs.com/consent-models), [US states](https://digitalprivacyregs.com/us-states), [calendar](https://digitalprivacyregs.com/calendar), and [register data](https://digitalprivacyregs.com/data/register.json) — secondary review leads and the dated source of the precautionary expansion.
10. [California Attorney General: CCPA](https://oag.ca.gov/privacy/ccpa) and [Canada OPC: online behavioural advertising](https://www.priv.gc.ca/en/privacy-topics/technology/online-privacy-tracking-cookies/tracking-and-ads/gl_ba_1112/) — examples of opt-out rights and conditional consent models.
11. [CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/) and [MaxMind GeoLite](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data/) — AWS usage costs and the distinction between a paid database product and free licensing options.
