# COOP by Default

## A Problem

When a web application opens another origin in a window, it obtains a JavaScript reference to that context that it can reach through to poke at various things. The opened context likewise receives a reference to its opener which provides similar access. This communication channel between the two windows enables attacks both at the web API level (`postMessage` vulnerabilities, navigation trickery, and [numerous side-channels](https://github.com/xsleaks/xsleaks/wiki/Browser-Side-Channels)), and at the process level (without OOPIF, there are clear issues, but even with OOPIF `document.domain` makes it difficult to put cross-origin-but-same-site documents into distinct processes).

These attacks are particularly interesting, as top-level navigation via newly opened windows will include `SameSite=Lax` cookies. This means that origins remain vulnerable to side-channel attacks that rely on user authentication (for example, [frame counting](https://github.com/xsleaks/xsleaks/wiki/Browser-Side-Channels#frame-count)) even in browsers that shift cookies to that more-secure default.

Web developers can defend themselves against these attacks by opting-into [Cross-Origin Opener Policy](https://html.spec.whatwg.org/multipage/origin.html#cross-origin-opener-policies), which can break the link between cross-origin windows by putting them into distinct browsing context groups. That is, when the clever developers at `secrets.example` assert a COOP, calling `window.open("https://secrets.example/")` from a cross-origin page will return a closed window handle, useless for side-channel attacks.

It's excellent that a defense exists, but it's unfortunate that the web places the burden of opting-into it on the victim. There are certainly good reasons for two cross-origin windows to communicate (SSO, payments, etc), but documents engaging in those flows should opt-in to the communication channel, rather than asking all other origins to opt-out.

## A Proposal

**In the absence of an explicit header, documents' [cross-origin opener policy](https://html.spec.whatwg.org/multipage/dom.html#concept-document-coop) should default to [`same-origin-allow-popups`](https://html.spec.whatwg.org/multipage/origin.html#coop-same-origin-allow-popups), rather than [`unsafe-none`](https://html.spec.whatwg.org/multipage/origin.html#coop-unsafe-none)**. This will have the effect of closing the window handle returned when a cross-origin document is opened via `window.open`, and disowning the opener of the newly created window.

Those few services which intend to maintain that relationship when opened may easily do so by explicitly declaring a cross-origin opener policy of `unsafe-none`. This will expose them to the risks discussed above, but they're choosing that risk intentionally in order to provide some service or obtain some information. They can reduce this risk by examining the incoming request, perhaps making decisions based upon request characteristics revealed in [Fetch Metadata](https://www.w3.org/TR/fetch-metadata/) headers, the user's cookies, and so on.

## FAQ

### Who needs to do work to opt back into the status quo?

The _openee_ needs to set a header. That is, if `https://a.example` opens `https://b.example` in a new window, `https://b.example` must send `Cross-Origin-Opener-Policy: unsafe-none` in order to maintain status quo behavior.

### The Cross-Origin-Opener-Policy header is ignored for non-secure documents, isn't it?

Yes. This change would require cross-origin documents that wish to establish communication channels with their opener to be served securely. That gives the opener confidence that they're talking to the service they expect to be talking to, and not to the network at large.

In the example above, `https://b.example` can opt into status quo behavior. `http://b.example` could not do the same. `b.example`'s administrators would be well-advised to redirect their users to HTTPS if they'd like to keep the relationship running.

Note that `http://a.example` would also be capable of establishing a communication channel with `https://b.example` if the latter explicitly opted-in. This puts the incentive in the right place: services that intend to be used widely via popups (consider SSO, payment, etc) need to be delivered securely.

### How many pages will this affect?

`window.open` is used on [~2.6% of page views](https://chromestatus.com/metrics/feature/timeline/popularity/475), which is a high-water mark for the impact. (Note that windows opened via `<a target=_blank>` do not create an opener relationship today in Safari and Firefox, and Chromium [is likely to follow suit](https://chromium-review.googlesource.com/c/chromium/src/+/1630010).)

Before shipping this change, we obviously need to gather better metrics on the number of window.open calls that result in a handle that's used for cross-origin communication, either from the opener to the openee, or vice-versa. My intuition is that it's a small percentage of that 2.6%, concentrated in semi-centralized services that we have a chance of poking at successfully.

### Why default to `same-origin-allow-popups`, and not `same-origin`?

`same-origin` is fairly strict, preventing pages from communicating with any cross-origin popup. This is a reasonable state for most applications to aim for, but it has the potential to break applications that expect to create popups that they can poke at. `same-origin-allow-popups` is weaker from an isolation perspective, but much more likely to be deployable without application developers noticing negative impacts; it places the burden on the somewhat-centralized popped-up services, rather than the applications that wish to use them.

### We don't do this for nested documents, why are new windows different?

Two answers:


1.  Cross-site nested documents are at reduced risk of attack, insofar as they aren't considered same-site. `SameSite=Lax` and `SameSite=Strict` cookies won't be sent, unlike top-level navigation resulting from `window.open`.

2.  You're right, friendly interlocutor! We _should_ default documents to asserting `Content-Security-Policy: frame-ancestors 'self'` in the absence of any other `frame-ancestors` or `X-Frame-Options` or `Cross-Origin-Resource-Policy` declaration. What a wonderful idea that would coincidentally substantially reduce risk for browsers without OOPIF! But that's another document for another day...
