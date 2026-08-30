# 1. Serve noirlore.com from a private bucket via CloudFront OAC

- **Status:** Accepted
- **Date:** 2026-08-30

## Context

The site originally used the S3 **website** endpoint (`noirlore.com.s3-website-us-east-1.amazonaws.com`) as a CloudFront custom origin. That arrangement has three properties worth naming:

1. The website endpoint has no TLS. The origin was configured `OriginProtocolPolicy: http-only`, so the CloudFront-to-S3 hop crossed the network unencrypted.
2. The website endpoint cannot be restricted to CloudFront. Serving it requires a bucket policy granting `s3:GetObject` to `Principal: "*"`, so the bucket was world-readable and every public-access block was off. The origin was reachable directly, bypassing CloudFront.
3. Error handling lived in the bucket's website configuration, whose `ErrorDocument` pointed at `404/index.html`. Hugo emits `404.html`. The key never existed, so every bad URL returned a bare S3 error page instead of a site page.

Point 3 was the visible symptom that started this work: two navigation links had pointed at unbuilt sections since launch, and the resulting raw S3 error made a healthy site read as down.

## Decision

Serve the bucket through a **REST origin** (`noirlore.com.s3.us-east-1.amazonaws.com`) with **Origin Access Control**, and make the bucket private.

- CloudFront signs origin requests with SigV4 as the `cloudfront.amazonaws.com` service principal.
- The bucket policy grants `s3:GetObject` to that principal alone, conditioned on `AWS:SourceArn` matching this distribution.
- All four S3 public-access blocks are enabled.
- The website-hosting configuration is deleted.
- 404s are handled by CloudFront custom error responses, not by S3.

## Consequences

**A REST origin does not resolve directory indexes.** The website endpoint mapped `/stories/` to `/stories/index.html` for free; the REST API treats it as a literal key and misses. A `viewer-request` CloudFront Function, `noirlore-dir-index`, restores it: URIs ending in `/` get `index.html` appended, and extensionless URIs get `/index.html`. Anything with a file extension passes through untouched.

This function is on the request path for **every** request. A bad edit takes the whole site offline, not one page. Test with `aws cloudfront test-function` against the DEVELOPMENT stage before publishing.

**Missing keys return 403, not 404.** The policy grants `s3:GetObject` but not `s3:ListBucket`, so S3 cannot distinguish "absent" from "forbidden" and returns `AccessDenied` for both. Mapping only 404 would leave real 404s showing as 403. Both codes map to `/404.html` with a `404` response code.

Granting `s3:ListBucket` would make S3 return a true 404, but it also lets the principal enumerate the bucket. Not worth it for a cosmetic distinction that the error mapping already handles.

**The deploy pipeline is unaffected.** CI writes with IAM user credentials, which are evaluated against that user's IAM policy rather than the bucket policy. Public-access blocks do not apply to authenticated principals. Verified by rerunning the workflow after the lockdown.

**Direct origin access is gone.** Both `https://s3.us-east-1.amazonaws.com/noirlore.com/*` and the old website endpoint now return 403. Every request goes through CloudFront, so caching, logging, and TLS policy have no bypass.

## Alternatives considered

**Point the website config's `ErrorDocument` at `404.html`.** One API call, and it would have fixed the visible 404 problem the same day. Rejected because it fixes only the symptom and leaves the bucket world-readable and the origin hop unencrypted.

**Keep the website origin and add a bucket policy restricted by a shared secret header.** A common workaround for restricting website endpoints. Rejected as strictly worse than OAC: the secret has to be rotated by hand, it still cannot encrypt the origin hop, and OAC is the supported mechanism for exactly this.

**Origin Access Identity instead of OAC.** OAI is the predecessor to OAC and is no longer recommended for new distributions. It does not support SSE-KMS origins or all regions. No reason to adopt it now.
