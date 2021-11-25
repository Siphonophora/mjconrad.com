---
layout: single
classes: wide
title:  "Serve your github.io site from a subdomain, using Google Domains"
date:   2020-11-02 09:00:00 -0400
permalink: blog/serve-your-githubio-site-from-a-subdomain-using-google-domains
header:
  teaser: /images/2020/serve-githubio-from-google-subdomain/teaser-500x300.png
---

Hosting your github.io site under a custom domain is a great option to make your project look more polished. This post will outline the steps to set up your project under a subdomain, using Google Domains. 

In the example we will be using, our goal is to switch hosting from `https://siphonophora.github.io/SqlDataCompare/` to `https://sqldatacompare.mjconrad.com/`. If you are looking for instructions on hosting at your root website, including the `www` subdomain, then take a look [at this post](https://dev.to/trentyang/how-to-setup-google-domain-for-github-pages-1p58).

## Starting Point

If you don't already have a github.io site setup, you can follow [this tutorial](https://swimburger.net/blog/dotnet/how-to-deploy-aspnet-blazor-webassembly-to-github-pages) to setup a Blazor WebAssembly site. That's where I started, but you should be able to follow this tutorial no matter what framework you are using for your site.

You should be starting with a site at [username].github.io/[projectName]

## Setup

At this point, you probably have a subdomain name selected. If you don't, standard URL naming rules apply, and the actual name deserves some consideration. I prefer legibility over brevity, but its a personal choice. What is important to know, is that **you can only configure a subdomain once**. What that means, is that if you have multiple github.io projects, you couldn't plan to host them all under a subdomain like this `apps.[site].com/[project]` with each project hosting on a different github.io project. On the other hand, something like `[project].apps.[site].com` should be ok, though I wouldn't add the extra complexity to the URL without reason.

To get everything configured, start by going to https://domains.google.com/ and selecting the DNS page. 

![](/images/2020/serve-githubio-from-google-subdomain/select_dns.png)

Next, we are going to add a `CNAME` to the `Custom resource records` section. If you haven't heard the term `CNAME` before, it is short for `Canonical Name`, which means this is the name of record for your site. The actual hosting location is effectively an implementation detail which your users and search engines don't need to be concerned with. For a `CNAME` record, the left entry is your subdomain, and the right is the domain where the site is actually being hosted. Note, that the domain is does not include the project name any more. We specify only `[username].github.io` as the domain name.

If you noticed that the `Syntetic Records` section allows configuration of subdomain forwarding, I want to confirm that you don't need to touch that section.

![](/images/2020/serve-githubio-from-google-subdomain/google_add_cname.png)

After you click `Add`, if you see an extra period after the domain name, it isn't a problem. 

![](/images/2020/serve-githubio-from-google-subdomain/google_extra_period.png)

Next, we open the settings in our GitHub project. 

![](/images/2020/serve-githubio-from-google-subdomain/github_settings.png)

Scroll to the GitHub Pages section and enter the custom domain, using the same subdomain configured in google. NOTE, we can't turn on `https` yet, because it will take about a day for GitHub to have a [Let's Encrypt](https://letsencrypt.org/) certificate issued for your custom domain. 

2021 Edit. Looks like the certificates get issued much faster now, so you may not need to wait very long.
{: .notice--success}

![](/images/2020/serve-githubio-from-google-subdomain/github_add_custom_domain.png)

Saving the custom domain has created a CNAME file, in the branch and folder you configured for GitHub Pages hosting.

![](/images/2020/serve-githubio-from-google-subdomain/github_cname.png)

At this point, Google and GitHub are in agreement that GitHub is serving your project at the subdomain you configured. In my experience, this change went into effect pretty quickly. 

You may have another step to get your site showing up at the new address if you initially had it running at GitHub Pages. During your initial setup, you may have had to configure the base href for you project so it was not `/` but `/[projectname]`. This is no longer needed. In my case, I was using a GitHub Action step to set up the href (from the [Blazor tutorial](https://swimburger.net/blog/dotnet/how-to-deploy-aspnet-blazor-webassembly-to-github-pages) mentioned above). Removing this was my last step to getting this project to show up at sqldatacompare.mjconrad.com.

![](/images/2020/serve-githubio-from-google-subdomain/remove_url_reqwrite.png)

**Don't Forget to return tomorrow to enable `https`. This is very important.** Browsers and search engines **may penalize your site** for being unencrypted. It could be marked as `unsafe` or may be ranked lower in search results.

![](/images/2020/serve-githubio-from-google-subdomain/github_https.png)


## Final State

Your site should now be hosted, with https, at your new subdomain. Happily though, the old URL is still going to work.

* https://sqldatacompare.mjconrad.com/ You can direct all traffic to the subdomain. 
* https://siphonophora.github.io/SqlDataCompare/ Any existing links which pointed to the page directly on github.io will continue to work just fine. 