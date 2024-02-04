---
title: "How this site works"
date: "2024-02-03"
lead: "How this site works"
disable_comments: false # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: true # Optional, enable MathJax for specific post
categories:
  - "meta"
tags:
  - "how-this-works"
---

**TL;DR**: [github.com/bpcreech/blog](https://github.com/bpcreech/blog) ⇨ [Hugo](https://gohugo.io/) (via Github actions) ⇨ [github.com/bpcreech.github.io](https://github.com/bpcreech.github.io) ⇨ Github pages. With DNS mapped via Google Domains (soon, RIP).

<!--more-->

## Tell me more for some reason

[This super awesome page](https://ruddra.com/hugo-deploy-static-page-using-github-actions/) by Arnab Kumar Shil shows how to:

1. Start from [blog content in source form](https://github.com/bpcreech/blog) (mostly [Markdown](https://www.markdownguide.org/)) in one Github repo.
2. [Configure](https://github.com/bpcreech/blog/blob/main/.github/workflows/publish.yaml) [Github Actions](https://github.com/features/actions) to automatically run the static site generator [Hugo](https://gohugo.io/) upon any commit and schlep that generatic content into [*another* Github repo](https://github.com/bpcreech/bpcreech.github.io).
3. From *there* the built-in [Github Pages](https://pages.github.com/) Github Actions automatically update bpcreech.github.io with that content.
4. We can also, thanks to Github Pages' [custom domains feature](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages), redirect traffic from bpcreech.com to bpcreech.github.io at the DNS layer.

    * One cool part about this which I wasn't expecting: Github pages automatically obtains a [Let's Encrpyt](https://letsencrypt.org/) certificate for domains you configure this way. So I didn't even have to generate a certificate!

6. Speaking of DNS, in my case the DNS layer is managed by Google Domains... [which is over time becoming SquareSpace domains](https://blog.pragmaticengineer.com/google-domains-to-shut-down).

Yay, highly-scalable web hosting for just the cost of $12/year for a custom domain! (Within, you know, [limits](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages) *including that you don't mind your source being public*.)

<p style="text-align: center;">
  
![A browser pointing at bpcreech.com, showing https enabled](/img/i-can-has-https.png) ![Cert details listing Let's Encrypt as the authority](/img/lets-encrypt.png)

</p>

## Alternative considered: Google Cloud Platform

*This is way too much info, just writing for my future reference I guess...*

An earlier version of this site, from way back in 2018, worked this way:

1. A Github web hook, which called,
2. A Google Kubernetes Engine (GKE) cluster,
3. Which ran [a tiny Go program](https://github.com/bpcreech/hugohook) which downloaded the repo and ran Hugo on it, and then...
4. Schlepped the generated files into Google Cloud Storage (GCS)

This worked! But several ways in which it wasn't as good:

### More complicated, more maintenance

Running your own GKE cluster can involve some maintenance and actual $ costs. Github Actions was launched [just a month after I originally set this up](https://techcrunch.com/2018/10/16/github-launches-actions-its-workflow-automation-tool) and is far easier to set up and maintain. Github Actions is much easier within the confines of what I want (including, e.g., that I don't have much in the way of expectations around availability which [has been problematic](https://www.githubstatus.com/history) on Github Actions, and the "build" operations I'm running are cheap enough to be free).

If I were to redo this *without* Github Actions I'd use [Google Cloud Run](https://cloud.google.com/run) (i.e., containerized serverless) along with [this fancy new Workload Identity Pool authn scheme](https://github.com/google-github-actions/auth?tab=readme-ov-file#direct-wif) in lieu of shared secrets. (I played with this too recently. It works! It's super cool! It's not worth it for this.)

### Unbounded costs

A GKE cluster and GCS bucket, like many things on Google Cloud, can generate unbounded cost if something goes really haywire. [Google Cloud does not offer any built-in way to set a max billing limit](https://stackoverflow.com/questions/27616776/how-do-i-set-a-cost-limit-in-google-developers-console). [This can bankrupt you overnight](https://news.ycombinator.com/item?id=25398148) unless you can successfully beg Google to forgive your bill. Probably okay if running a company which has an ops team to pay attention to such things. Definitely not worth the risk for a personal blog which I sometime forget exists!

Now, you can set up a [Cloud Function which checks and disables billing](https://cloud.google.com/billing/docs/how-to/notify). I played with this. It's not terribly hard to get working, but you need to test it carefully, because the consequencies if it fails are, again, [dire](https://news.ycombinator.com/item?id=25398148)). That makes me nervous, so no thanks, for now.

### The cheap flavor of serving a static site from Cloud Storage is ~deprecated because no HTTPS

Google [used to host instructions](https://web.archive.org/web/20180112010509/https://cloud.google.com/storage/docs/hosting-static-website) for serving a static site from Google Cloud Storage (GCS) by simply pointing your domain's `A` and `AAAA` records to a GCS VIP. The servers behind the GCS VIP, which presumably got requests for a gazillion different buckets, would then use some form of [vhosting](https://en.wikipedia.org/wiki/Virtual_hosting) to identify which bucket to serve up.

This is, presumably, what Github pages do today.

The problem was that [this only worked for plaintext `http` (not `https`)](https://web.archive.org/web/20170327185149/https://cloud.google.com/storage/docs/static-website#https). It looks like Google never did the magic trick that Github Pages do today of dynamically provisioning a cert (using Let's Encrypt). You could argue that this silly blog doesn't need `https` but it's just [not the way winds are going](https://blog.chromium.org/2023/08/towards-https-by-default.html), and it's not surprising that Google silently removed these old vhosting-oriented instructions.

They *could* have gone with the Let's Encrypt path, but instead the [new host-your-site-from-GCS instructions](https://cloud.google.com/storage/docs/hosting-static-website) go with a more complex path of:

1. Allocating a static VIP (presumably both IPv4 and IPv6),
2. Obtaining and generating your own cert, and
3. Setting up the Google Cloud Load Balancer pointing that VIP to your bucket using your cert.

This is more flexible and powerful, but it's more complicated. The fact that it uses a static VIP means you need to pay for an ever-diminishing and thus [ever-more-costly resource](https://cloud.google.com/vpc/network-pricing#ipaddress). (As of this writing, unused IPv4 addresses cost $113, and AFAICT ones mapped to GCS buckets are free?! I assume that will change eventually as [availability dwindles](https://ipv4.potaroo.net/).)

I suppose that once IPv4 is sufficiently exhausted, after the ensuing chaos, everyone's crappy ISP will finally support IPv6 it will be more reasonable to just generate free static IPv6 VIPs (forgoing IPv4), and then dispense with server-side tricks like vhosting to map multiple sites onto one address. In the meanwhile, "I just want to serve 4 TB from GCS" is a slightly more complicated and potentially not-quite-free.
