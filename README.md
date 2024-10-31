# Subresource Reporting

Complex web application often need to keep tabs of the subresources that they download, for security purposes.

In particular, upcoming industry standards and best practices (e.g.
[PCI-DSS v4](https://east.pcisecuritystandards.org/document_library?category=pcidss&document=pci_dss) - 
[context](https://docs.google.com/document/d/1RcUpbpWPxXTyW0Qwczs9GCTLPD3-LcbbhL4ooBUevTM/edit?tab=t.0#heading=h.dzquzu6onmmy))
require that web applications keep an inventory of all the scripts they download and execute.

This document proposes a new web platform feature on top of the
[Reporting API](https://www.w3.org/TR/reporting-1/),
that would enable web developers to create and maintain such inventories in a secure manner.

## Problem

Web developers load many different script assets to their sites, and those scripts can then load other assets.
Some of those assets are versioned and their content's integrity can be validated using
[Subresource Integrity](https://w3c.github.io/webappsec-subresource-integrity/)
or using
[Content Security Policy hashes](https://www.w3.org/TR/CSP3/#grammardef-hash-source).
But other assets are dynamic, ever-green scripts that can be updated by their provider at any moment.
The web platform has no means of validating the integrity of such scripts, neither in reporting nor in enforcement mode.

At the same time, 
[upcoming security standards](https://docs.google.com/document/d/1RcUpbpWPxXTyW0Qwczs9GCTLPD3-LcbbhL4ooBUevTM/edit?tab=t.0#heading=h.dzquzu6onmmy)
require web developers to maintain an up to date inventory of all scripts that execute in the context of their payment page documents,
and have a mechanism to validate their integrity.

In the absence of better mechanisms, developers and merchants will need to settle for lower fidelity security guarantees — e.g. offline hash verification through crawling.
Such mechanisms leave a lot to be desired in terms of their coverage, while at the same time add a lot of implementation complexity. We need a better path.

## Proposal

A new Reporting API feature could be used to send reports of all scripts executed in the context of the relevant document,
including their URLs and their hashes (for CORS-enabled resources).

That would enable developers to set up endpoints that collect these reports, and process them to maintain an up to date and accurate
inventory of scripts and their integrity for relevant pages.

### Flow
Developers can set the following headers on their navigation responses:
`Reporting-Endpoints: subresources="https://example.com/reports"`
`Subresource-reporting: script=subresources`

The `Subresource-reporting` header would be defined as a
[Dictionary](https://www.rfc-editor.org/rfc/rfc8941#name-dictionaries),
to enable future extensibility in case we'd want to report subresources beyond scripts.

Each loaded and executed script would then
[generate and queue a report](https://www.w3.org/TR/reporting-1/#generate-and-queue-a-report)
with the resource's URL and, if the resource was requested with a "cors"
[mode](https://fetch.spec.whatwg.org/#concept-request-mode),
its integrity
[digest](https://w3c.github.io/webappsec-subresource-integrity/#digest).

That would eventually send a report that looks something like the following:
```
POST /reports HTTP/1.1
Host: example.com
...
Content-Type: application/reports+json

[{
  "type": "subresource",
  "age": 12,
  "url": "https://example.com/",
  "user_agent": "Mozilla/5.0 (X11; Linux i686; rv:132.0) Gecko/20100101 Firefox/132.0",
  "body": {
    "url": "https://example.com/main.js",
    "digest": "sha256-badbeef"
  }
}]
```

If multiple script resources would be queued before reports are sent (at a user agent defined time), the user agent
[will serialize the entire list of reports](https://www.w3.org/TR/reporting-1/#try-delivery) and send them in a single report.

## Considered alternatives

### Resource-timing

The Resource Timing API can be used to gather up all the URLs of script resources in a document today.

It could also be extended to provide an integrity hash for CORS resources. 

However, that is not ideal because:
* Currently browsers don’t calculate these hashes, so starting the calculate them for all scripts in all documents could introduce some overhead.
  Therefore, adding hashes to resource timing may require some opt-in mechanism.
* Malicious scripts on the site can tamper with Resource Timing data and obfuscate their presence on the page (by e.g. reporting known-good hashes instead of their own).


### CSP `require-sri-for` + hash reporting

Initially it looked like the abandoned
[`require-sri-for`](https://udn.realityripple.com/docs/Web/HTTP/Headers/Content-Security-Policy/require-sri-for)
CSP feature may be a good fit here, when combined with report-only mode.

But at a second look, CSP enforcement and reporting happens at request-time, the current implementation does not have access the the resource hashes.
That's doubly true in enforcement mode, where the resources are never downloaded (and hence the browser never knows the hash).

Including hash reporting only in report-only mode felt hacky, and would've required significant changes to the CSP reporting implementation.

### CSP report-only mode + hash reporting

Similarly to the previous section, we could consider adding hash reporting to `script-src` CSP directives. They would suffer from all the issues that `require-sri-for` would suffer from.
On top of that, they may collide with actual CSP policies that the site would want to deploy.

## Security & Privacy considerations

This proposal doesn't expose new information when it comes to URLs - URLs are already exposed in Resource Timing and Service Workers,
and developers can use the `initiatorType` or `Request.destination` respectively to get that information, albeit in less-secure and more complex ways.

Resource hashes are not currently exposed, but we plan to expose them only for CORS-enabled resources, that the document can already fully read.
That means that developers can already fetch those resources and calculate their hashes on their own. (again, with added complexity)

## Open questions

* Bikeshedding - Should we limit this to scripts only and call it Script Reporting?
  - Leaving room for future extensibility if needed feels better.
* Do we want visibility to ReportingObserver?
  - Any use cases that need it?
* Would the CORS requirement prevent popular scripts from having their hashes collected?
  - If so, do we need a complementary Document-Policy that enforces CORS for all subresources?
