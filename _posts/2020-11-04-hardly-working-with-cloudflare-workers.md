---
title: Hardly working with Cloudflare Workers
author: Malthe Jørgensen
---

<!-- Possible titles:
Cloudflare workers are hard to work with
Working with Cloudflare Workers
Hardly working with Cloudflare Workers
-->

_Note: The team behind Notifly also runs [Eduflow](https://www.eduflow.com) and [Peergrade](https://www.peergrade.io)._

## Introduction

This is the story of me trying to replace a simple NGINX reverse proxy (plus some basic redirects) with a Cloudflare Worker.

Our old landing page is a Wordpress blog hosted on WPEngine. Historically, this has always been set up behind an NGINX reverse proxy serving at [peergrade.io](http://peergrade.io) and [www.peergrade.io](http://www.peergrade.io). The reverse proxy was needed for doing various redirects outside of Wordpress and doing some cookie trickery to redirect to [app.peergrade.io](http://app.peergrade.io) if the session cookie for the app was present.

The reverse proxy is hosted on DigitalOcean and is the only thing we have hosted there, so I wanted to get rid of it. We already use Cloudflare and so I thought "this would be a good test to try out Cloudflare Workers". And less infrastructure is better, right?

## The good parts

Getting set up with `wrangler` – the CLI for Cloudflare Workers – was a breeze. It gives you a webpack setup out of the box which allowed me to install NPM packages and use them without any extra work on my part. I eventually downgraded to a setup without webpack (called "javascript" in `wrangler`) – since I ended up not needing any packages.

The vanilla Javascript setup allows you to "live edit" the worker at `https://dash.cloudflare.com/<account-id>/workers/edit/<worker-slug>`

Here you can edit and run the updated script without saving and deploying the worker, allowing for a very fast and easy "edit-compile-run" loop.

Another cool thing is that you can change the URL in the small "browser" on the page to your liking – this is very useful for testing out proxies and other things that depend on the domain name or precise URL being sent to the worker. The debugger part of the UI is also incredibly useful but does have a tendency to disconnect from time to time.

## Page Rules vs. Workers

In a classic setup you'll usually have a couple of redirects alongside your reverse proxy – and so do we. We use [www.peergrade.io](http://www.peergrade.io) as our canonical domain so we redirect peergrade.io to www.peergrade.io and we redirect http:// to https://.

This can be set up easily in Cloudflare by adding a couple of redirects in your Page Rules.

However, redirects from page rules are applied after any worker on the same URL. Since my worker's default action is to reverse proxy, the redirect page rule will never be hit.

Annoyingly, this isn't clearly described in the docs and you'll have to find [this forum](https://community.cloudflare.com/t/cf-workers-and-rate-limiting-firewall-rules-bot-management/132164/3) post from the official Cloudflare forum to know that. The forum post notes that "security-related ones will run before" – but which ones are those? (All respect to Kenton Varda who wrote the post and is the main architect behind Cloudflare Workers. Cloudflare Workers *are* very very cool, but they are also a bit more quirky than I'd like at the moment)

In order to preserve these redirects, I'll have to manually write them in the worker code (or relay the URLs that need to redirect to Cloudflare itself – which is basically the same amount of code). 

<!-- 
- An aside:

    Apparently *Always Use HTTPS* is such a "security-related" page rule, even though it's basically an http:// to https:// redirect. Cloudflare even admits to that [in the docs](https://support.cloudflare.com/hc/en-us/articles/204144518-SSL-FAQ#h_a61bfdef-08dd-40f8-8888-7edd8e40d156). 

    Cloudflare Page Rules allows you to set up multiple rules for a single URL-pattern, but then only allows you to use that pattern once. However, *Always Use HTTPS* is special and doesn't allow any other rules once it's used on a URL-pattern. This means if you want *Automatic HTTPS Rewrites* on top of *Always Use HTTPS* you have to specify 2 rules:

    1. www.peergrade.io – *Always Use HTTPS*
    2. [https://www.peergrade.io](https://www.peergrade.io) – *Automatic HTTPS Rewrites*
-->

The same thing goes for cache rules. I had previously been using a page rule to aggressively cache static assets and user-uploaded content served from Wordpress. That now has to be written inside the worker as well.

Page rules have an internal ordering that you can set. Rules that match the given URL are executed in order – so that if two redirect rules match the URL, the first one in the ordering will be used. It would be *really nice* if workers could be added to the same list – that would mean I could put the redirects and cache rules before my worker and much more easily handle this scenario. *In principle* this would be easy if all the built-in page rules were reimplemented as workers, but there's probably legacy behaviors and tie-ins to the rest of the stack that makes that impossible or at least non-trivial. (Still hoping for a future update on this 🤞🏻)

## The reverse proxy

Back to the main task at hand – we're implementing a simple reverse proxy and that happens to be [one of the examples](https://developers.cloudflare.com/workers/examples/bulk-origin-proxy) in the Cloudflare Worker docs. However, getting it set up myself I quickly ran into issues with redirect loops and cases where my origin would redirect for seemingly no reason. To be fair, proxying can be tricky to get right since it's hard to test properly before rollout, and on top of that you have DNS propagation and caching, which means there might be timing issues. But even with that, it seemed extra tricky with Cloudflare Workers.

On closer inspection, the example from the Cloudflare docs seems to defy reasoning. The incoming request in the example must have the header `Host: google.yourdomain.com` in order for it to match the Google entry in `ORIGINS`. I was able to confirm as much by inspecting the incoming request in the Cloudflare worker debugger. That incoming request is then relayed directly to `www.google.com`. Let's try that ourselves:

 `curl -H 'Host: google.yourdomain.com' https://www.google.com`

The response we get is a 404 page (which makes sense since the host doesn't match). However, the Cloudflare worker doesn't get a 404 – it renders the familiar Google search frontpage. Something must be happening behind the scenes. That something is what I call "The Web Platform" part of Cloudflare Workers.

## The Web Platform

Cloudflare Workers uses Chrome's V8 as its execution engine and this also sets the context in which your script is run.

The available API is a very small subset of [The Web Platform](https://platform.html5.org/) (the Javascript API available in modern browsers) -- specifically Ecmascript/Javascript itself, plus `Fetch`, `URL`, and `Blob`. I believe Cloudflare chose this API because it melds well with V8, but also because web devs will be familiar with those APIs. But how familiar are you *really* with `fetch`, `Request`, and `Response`? (all part of the `Fetch`-spec)  
I don't think I actually knew the `Request` and `Response`-objects in any detail before using Cloudflare Workers – having gotten along just fine with variations of 

    fetch('http://example.org', { options }).then((r) => r.json())

plus some error handling on top for many years. 

When working with Workers what you'll mostly be doing is to manipulate the incoming `Request`-object  and pass it on to `fetch`, or manipulate the outgoing `Response`-object and passing that on to Cloudflare's handler. Have you ever manually created a `Request`-object in the browser? I haven't. The reason this gets complicated is the fact that the spec for `fetch` itself is very "loose". For example, `fetch` can take either a `Request`-object or a simple Javascript object that just looks a lot like a `Request`-object as its argument -- and it not really clear what differences between the two are.
`fetch` also allows passing a `Request`-objects as both its first and second argument `fetch(Request(...), Request(...))` -- good luck trying to figure out what that does!

If we go back to the example from the Cloudflare docs -- what's going on "behind the scenes" in our proxy example from earlier is that you can't change the `Host`-header when doing a `fetch`. This makes a lot of security sense in the browser where `fetch` normally lives, but it's quite normal behavior for a reverse proxy and actually something I was doing in my NGINX setup in order to have WPEngine respond with the right content. It's not a behavior you've ever needed or thought about when using `fetch` in the browser.
The server is just a very different environment than the browser. The browser Javascript API is not built with server functionality in mind, and it ends up being a hamstring when working with Cloudflare Workers.

## Maybe, maybe, maybe...?

A bunch of forum posts on community.cloudflare.com talk about this issue

* **Only available on the Enterprise plan?** [This forum post][1] describes that setting the `Host` -header in workers is not possible. Followed up with [a later answer][2] that it's possible but only for Enterprise accounts.
 [This other post][3] says the same.
- **Kenton Varda to the rescue** In response to [this post][4] Kenton Varda actually extends Cloudflare Workers with the `cf.resolveOverride`-flag on the `Request`-object,
  which should allow at least part of the reverse proxy setup to work.
  Unfortunately, to explain the new feature the post just links to the top-level URL of the documentation for Cloudflare Workers – which currently doesn't
  describe how  `cf.resolveOverride` works and how to use it.
- **The missing documentation** [This older post][5] seemingly cites documentation that no longer exists! :(  
   I have been unable to find any meaningful documentation of `cf.resolveOverride` outside of the community forum, and I was unable to have it allow me to switch the `Host`-header.

[1]: https://community.cloudflare.com/t/override-host-header-using-workers/73434/2
[2]: https://community.cloudflare.com/t/override-host-header-using-workers/73434/5
[3]: https://community.cloudflare.com/t/reverse-proxy-using-page-rules/47836/16
[4]: https://community.cloudflare.com/t/not-possible-to-override-the-host-header-on-workers-requests/13077/7
[5]: https://community.cloudflare.com/t/different-hostname-with-same-origin-in-workers/16662/12

## The final nail in the coffin

For a brief moment I actually thought my setup was working, but it only "looked" like it was working due to the following sequence of events:

- A request for `www.peergrade.io` would hit the worker
- The worker would then do a request to `peergrade.wpengine.com`
- Wordpress/WPEngine would then respond with a redirect to `www.peergrade.io` since the `Host`-header is incorrect
- Cloudflare by default then follows that redirect and makes a new request to `www.peergrade.io`.
  Since Cloudflare is the host of `www.peergrade.io` you'd think we'd hit infinite recursion here.
  But magically, it doesn't just enter the worker script again – it knows (somehow) it has to go further down the Cloudflare stack.
  Since the DNS A-record in Cloudflare still had the IP of the DigitalOcean instance, that final fetch would simply fetch the page from the old proxy server which worked as it always had 🤦🏻



<!--
Another example of this "familiar but unfamiliar" API is when I was trying to inspect the session cookie: I had to do a base64 decode into a `Uint8Array` (in order to do a zlib decompression). The function available for decoding base64 is `atob` which you may know from the browser.

However, in order to get the actual binary data you'll have to do this Javascript incantation:

```jsx
const weirdstr = atob(cookiestr);
const bytearray = new Uint8Array(new ArrayBuffer(weirdstr.length));

for (let i = 0; i < weirdstr.length; i++) {
  bytearray[i] = weirdstr.charCodeAt(i);
}
```

Again, this isn't Cloudflare's fault per se, but they're inheriting a bad choice from The Web Platform where they could have done something else. That bad choice becomes accentuated by the fact that most workers need to implement something that is basically backend or proxy server behavior, which by now you can see The Web Platform really isn't set up for. 

Similarly, you'll inherit this weird quirk directly from the browser Javascript engine:

```jsx
console.log(btoa('汉字'))
// The above raises a DOMException in your Cloudflare Worker with the
// following message:
// "btoa() can only operate on characters in the Latin1 (ISO/IEC 8859-1) range."
```

Yes yes, there's some sense to this – Javascript strings are UTF-16 and that's why this example doesn't work. But take a look at Node.js where `btoa` and `atob` are not available – Node.js has a much better answer to many of these problems.

Lastly, since many things are iterables or DOM-objects, you won't get anything useful out of console logging `request.headers`, `request.headers.keys()`, `request.headers.values()`, `request.headers.entries()`. This wouldn't be a problem if the `request`-object was fully inspectable in the debugger but nothing shows up when you open up `request.headers`.
The solution to this is just `console.log([...request.headers])`.
-->

## Conclusion

Overall, Cloudflare Workers are really cool and the tooling around them is pretty great. But I do feel like it was an unfortunate choice to adopt The Web Platform
instead of using parts of the Node standard library or a different, more server-oriented API. Lastly, while the documentation feels fairly complete and fleshed out -- the fact
that the answers on the forum tell 2-3 different stories about whether it's possible to change the `Host`-header means that it's something that is just begging to be
clarified in the docs.

And yes, I know the [Notifly docs](https://docs.notifly.io/) definitely aren't as great as Cloudflare Workers'. Luckily, the surface area is also a lot smaller 😅

