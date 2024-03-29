---
title: "How this site works"
date: "2024-02-03"
lead: "It's a series of tubes"
disable_comments: false # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: true # Optional, enable MathJax for specific post
categories:
  - "meta"
tags:
  - "how-this-works"
---

**TL;DR**: [github.com/bpcreech/blog](https://github.com/bpcreech/blog) ⇨
[Hugo](https://gohugo.io/) (via Github actions) ⇨
[github.com/bpcreech.github.io](https://github.com/bpcreech.github.io) ⇨ Github
pages. With DNS mapped via Google Domains (soon,
[RIP](https://blog.pragmaticengineer.com/google-domains-to-shut-down)).

<!--more-->

## Tell me more for some reason

[This super awesome page](https://ruddra.com/hugo-deploy-static-page-using-github-actions/)
by Arnab Kumar Shil shows how to:

1. Start from [blog content in source form](https://github.com/bpcreech/blog)
   (mostly [Markdown](https://www.markdownguide.org/)) in one Github repo.
2. [Configure](https://github.com/bpcreech/blog/blob/main/.github/workflows/publish.yaml)
   [Github Actions](https://github.com/features/actions) to automatically run
   the static site generator [Hugo](https://gohugo.io/) upon any commit and
   schlep that generatic content into
   [_another_ Github repo](https://github.com/bpcreech/bpcreech.github.io).
3. From _there_ the built-in [Github Pages](https://pages.github.com/) Github
   Actions automatically update [bpcreech.github.io](https://bpcreech.github.io)
   with that content.

   ```goat
   +--------------------------+     .--------------------------.
   |                          |    | Github actions:            |
   | github.com/bpcreech/blog +--->| Format markdown, Run Hugo, +
   |                          |    | Commit to next...          |
   +--------------------------+     '-------------+------------'
                                                  |
                                                  v
    .------------------.   +----------------------------------------+
   | Github actions:    |  |                                        |
   | Upload             |<-+ github.com/bpcreech/bpcreech.github.io |
   | Deploy             |  |                                        |
    '---------+--------'   +----------------------------------------+
              |
              v
   +--------------------+
   | bpcreech.github.io |
   +--------------------+
   ```

4. We can also, thanks to Github Pages'
   [custom domains feature](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages),
   redirect traffic from [bpcreech.com](https://bpcreech.com) to
   [bpcreech.github.io](https://bpcreech.github.io) at the DNS layer.

   - One cool part about this which I wasn't expecting: Github pages
     [automatically obtains](https://docs.github.com/en/pages/getting-started-with-github-pages/securing-your-github-pages-site-with-https)
     a [Let's Encrypt](https://letsencrypt.org/) certificate for domains you
     configure this way. So I didn't even have to generate a certificate!

5. Speaking of DNS, in my case the DNS layer is managed by Google Domains...
   [which is over time becoming SquareSpace domains](https://blog.pragmaticengineer.com/google-domains-to-shut-down).

Yay, highly-scalable web hosting for just the cost of $12/year from Google
Domains for a custom domain! (Within, you know,
[limits](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages)
_including that you don't mind your source being public_.)

<div>

<style>
table, td, th {
   border: none!important;
}
</style>

| ![A browser pointing at bpcreech.com, showing https enabled](/img/i-can-has-https.png) | ![Cert details listing Let's Encrypt as the authority](/img/lets-encrypt.png) |
| -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| <p style="text-align: center;">Hooray, `https` works!</p>                              | <p style="text-align: center;">Thanks, Let's Encrypt!</p>                     |

</div>

## Alternative considered: Google Cloud Platform

_This is way too much info, just writing for my future reference I guess..._

An earlier version of this site, from way back in 2018, worked this way:

1. A Github web hook, which called,
2. A Google Kubernetes Engine (GKE) cluster,
3. Which ran [a tiny Go program](https://github.com/bpcreech/hugohook) which
   downloaded the repo and ran Hugo on it, and then...
4. Schlepped the generated files into Google Cloud Storage (GCS)

This worked! But several ways in which it wasn't as good:

### More complicated, more maintenance

Running your own GKE cluster can involve some maintenance and actual $ costs.
Github Actions was
[launched in October 2018](https://techcrunch.com/2018/10/16/github-launches-actions-its-workflow-automation-tool)
just a month after I originally set this up. It's far easier to set up and
maintain within the confines of what I want (including, e.g., that I don't have
much in the way of expectations around availability which
[has been problematic](https://www.githubstatus.com/history) on Github Actions,
and the "build" operations I'm running are cheap enough to be free).

If I were to redo this _without_ Github Actions I'd use
[Google Cloud Run](https://cloud.google.com/run) (i.e., containerized
serverless) along with
[this fancy new Workload Identity Pool authn scheme](https://github.com/google-github-actions/auth?tab=readme-ov-file#direct-wif)
in lieu of shared secrets. (I played with this too recently. It works! It's
super cool! It's not worth it for this.)

### Unbounded costs

A GKE cluster and GCS bucket, like many things on Google Cloud, can generate
unbounded cost if something goes really haywire.
[Google Cloud does not offer any built-in way to set a max billing limit](https://stackoverflow.com/questions/27616776/how-do-i-set-a-cost-limit-in-google-developers-console).
[This can bankrupt you overnight](https://news.ycombinator.com/item?id=25372336)
unless you can successfully beg Google to forgive your bill. Probably okay if
running a company which has an ops team to pay attention to such things.
Definitely not worth the risk for a personal blog which I sometime forget
exists!

Now, you can set up a
[Cloud Function which checks and disables billing](https://cloud.google.com/billing/docs/how-to/notify).
I played with this. It's not terribly hard to get working, but you need to test
it carefully, because the consequencies if it fails are, again,
[dire](https://news.ycombinator.com/item?id=25372336). That makes me nervous, so
no thanks, for now.

### The simple & cheap flavor of serving a static site from Cloud Storage is ~deprecated because no HTTPS

Google
[used to host instructions](https://web.archive.org/web/20180112010509/https://cloud.google.com/storage/docs/hosting-static-website)
for serving a static site from Google Cloud Storage (GCS) by simply pointing
your domain's `A` and `AAAA` records to a GCS VIP. The servers behind the GCS
VIP, which get requests for a gazillion different buckets, would then use some
form of [vhosting](https://en.wikipedia.org/wiki/Virtual_hosting) to identify
which bucket to serve up.

This is, presumably, similar to how Github pages work today.

The problem was that
[this only worked for plaintext `http` (not `https`)](https://web.archive.org/web/20170327185149/https://cloud.google.com/storage/docs/static-website#https).
It looks like Google never did the magic trick that Github Pages do today of
dynamically provisioning a cert (using Let's Encrypt). You could argue that this
silly blog doesn't need `https` but that's
[not on the right side of history](https://blog.chromium.org/2023/08/towards-https-by-default.html),
and so it's not surprising that Google has removed these old vhosting-oriented
instructions.

They _could_ have gone with a Github-Pages-like Let's Encrypt path, by setting
up a place to declare a desire for Google to go provision a cert for you, but
instead, the
[new host-your-site-from-GCS instructions](https://cloud.google.com/storage/docs/hosting-static-website)
go with a more complex path of:

1. Allocating your own static VIP (probably both IPv4 and IPv6),
2. Obtaining your own cert, and
3. Setting up the Google Cloud Load Balancer pointing that VIP to your bucket
   using your cert.

This is more flexible and powerful (way more knobs to play with on both the cert
and the load balancer!), but it's more complicated. The fact that it uses a
per-site static VIP means you need to pay for an ever-diminishing and thus
[ever-more-costly resource](https://cloud.google.com/vpc/network-pricing#ipaddress).
(As of this writing, unused IPv4 addresses cost $113 / year, and AFAICT ones
mapped to GCS buckets are free?! I assume that will change eventually as
[availability dwindles](https://ipv4.potaroo.net/).)

I suppose that once IPv4 is sufficiently exhausted, after the ensuing chaos,
everyone's crappy ISP will finally support IPv6. Finally, it will be more
reasonable to just generate free static IPv6 VIPs (forgoing IPv4), and then
dispense with server-side tricks like vhosting to map multiple sites onto one IP
address. In the meanwhile, "I just want to serve 4 TB from GCS" is slightly more
complicated and potentially not-quite-free.
