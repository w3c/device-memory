# Device Memory

## Motivation & Use-cases
“one-size-fits-all” web experience does not work in a world with widely varying device capabilities. Web apps that delight users on high end devices can be unusable on low end devices, particularly in emerging markets.
To support a diverse web ecosystem that is inclusive of users on low end devices, apps need to be able to tailor their content to the device constraints. This allows users to consume content suited to their device, and ensures they have a good experience and fewer frustrations.

Developers are interested in the “device-class” for the following known use-cases:
1. Serve a “light” version of the site or specific components, for low end devices. Egs:
- Google’s "search lite" is a 10KB search results page used in EM. 
- Serve a light version of video player in Facebook
- Serve lightweight tile images in Google Maps
2. Normalize Metrics: analytics need to be able to normalize their metrics against the device-class. 
For instance, a 100ms long tasks on a Pixel is a more severe issue vs. on a low end device.

Device memory is an interesting signal in this context. Low memory devices devices (under 512MB, 512MB - 1GB) are widely used in emerging markets. Chrome telemetry indicates large number of OOM (out-of-memory) crashes on foreground tabs on these devices. In this case, serving a lite version not only improves the user experience, it is necessary for the site to be usable at all (as opposed to crashing due to memory constraint).

## Proposal
We propose a new HTTP Client Hints header and a web exposed API to surface device capability for memory (RAM). The mechanism should be extensible to other device capabilities such as CPU i.e. number of cores, clock speed etc.
A Client Hints header will enable the server to deliver appropriate content, eg. a “lite” version of the site.
The JS API will be a convenience for analytics reporting and may be used to make runtime decisions eg. using more storage vs. making additional requests, requesting appropriate resources from the server etc.

ASIDE: the JS API for CPU Cores is already available via hardwareConcurrency API

### The Header
Proposed Client Hints Header for memory: `Device-Memory`\
`Device-Memory : <value>`\
where `<value>` is an approximation of the amount of ram in GiB (floating point number).\
The `<value>` is calculated by using the actual device memory in MiB then rounding it to the nearest number where only the most signicant bit can be set and the rest are zeros (nearest power of two). Then diving that number by 1024.0 to get the value in GiB.

An upper bound and a lower bound should be set on the list of values. While implementations may choose different values, the recommended upper bound is 8GiB and the recommended lower bound is 0.25GiB (or 256MiB).

The following table illustrates some examples:

| Actual in MiB | Rounded in MiB | Reported in GiB |
|---------------|----------------|-----------------|
| 500           | 512            | 0.5             |
| 512           | 512            | 0.5             |
| 768           | 512            | 0.5             |
| 1000          | 1024           | 1               |
| 1793          | 2048           | 2               |

A full list of possible values is as follows:
0.25, 0.5, 1, 2, 4, 8

#### Why separate header and rounding?
HTTP caching doesn't deal well with mixed value headers, therefore separate headers are recommended. Also, rounding down to power of two enables caching and mitigates fingerprinting.

#### When is the header sent? 
Client Hints cannot be used to enable providing network performance data on the first request, however this is being actively addressed with [this proposal](https://github.com/httpwg/http-extensions/issues/306#issuecomment-283549512).
The header is sent after an explicit per-origin opt-in via Client Hints mechanism. The following new hint will be added: `Accept-CH: Device-Memory`

For background, [Client Hints](http://httpwg.org/http-extensions/client-hints.html) provides a set of HTTP request header fields, known as Client Hints, to deliver content that is optimized for the device. In that sense using Client Hints is a great fit for this proposal.
Client Hints recently addressed a [significant limitation in spec](https://github.com/httpwg/http-extensions/issues/306#issuecomment-283549512): opt-in can now be persisted across browser restarts using `max-age` in header.

#### Usage Guidance
Servers can advertise support for Client Hints using the Accept-CH header field or an equivalent HTML meta element with http-equiv attribute
```
  Accept-CH = Device-Memory
```

The Memory request header field is a number for the client’s device memory i.e. approximate amount of ram in GiB.
eg. 512 MiB will be reported as:
```
    Device-Memory: 0.5
```

Servers can use this header to customize content for low memory device eg. serve light version of the app or a component such as a video player.

NOTE: This header will not be immediately available in all browser. Do not assume presence of the header, and provide a graceful fallback for browsers that don't support the header.

### The web exposed API
We propose adding the following API to navigator: `navigator.deviceMemory`
which returns number of GiB of ram (floating point number) rounded down to the nearest power of two (same as the header).

## Why not surface Device Class directly?
While exposing a composite “Device Class” would be useful for developers, it has a number of challenges: it’s hard to specify in a future-proof manner as it is constantly changing, it requires significant and ongoing work (constantly update algorithm OR classify known devices), it is difficult to reach agreement amongst vendors, and come up with something that works for all web sites etc.
While this is something worth considering down the road, we think we can get most of the benefit by exposing a couple of specific signals device memory and device CPU cores. In particular device memory is a reasonable proxy for device class.

## Security & Privacy
The client hint header and API will only be available on HTTPS secure connections.

See [Security section in Client Hints](http://httpwg.org/http-extensions/client-hints.html#security-considerations).
Requiring per-origin opt-in with Accept-CH restricts when the header is advertised.
Restricting to a ceiling value (rounded to 2 most signicant bits), as opposed to exact value, reduces fingerprinting risk. 

NOTE: Device identification is already possible and rampant today, based on UA string. E.g. https://deviceatlas.com/products/web

## Relevant Links

* [Facebook's device year class](https://github.com/facebook/device-year-class)
* [HTTP Client Hints](https://github.com/igrigorik/http-client-hints)
* [HTTP Client Hints User Guide](https://developers.google.com/web/updates/2015/09/automating-resource-selection-with-client-hints)
* [Similar proposal for Network Speed](https://github.com/WICG/netinfo/issues/46)
* [NavigatorConcurrentHardware API](https://developer.mozilla.org/en-US/docs/Web/API/NavigatorConcurrentHardware)


