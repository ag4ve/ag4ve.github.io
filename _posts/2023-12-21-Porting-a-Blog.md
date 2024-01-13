---
pin: true
layout: post
title: "Porting a Blog"
date: 2023-12-21
author: shawn
categories:                                         
  - thoughts
tags:
  - idea
image:
  path: /static/img/WorldWideWebAroundWikipedia.png
---

## Preface

This is not going to be a very technical blog. If you are looking for normal Linux systems reading, this isn't it. What I want to cover is why I moved off of a hosted blog platform. So I've kept the technical details to a minimum.

There are pros and definite cons to this move. Namely, I wrote this as a non-technical blog post because I thought I could outline steps that a non-technical person could follow to get here. While it doesn't take much knowledge to write markdown, managing a git repo (especially with changes to GitHub Actions that stop updates from deploying), isn't as trivial as washing dishes. That said, If you are technical, hopefully this blog site may give you inspiration to create your own Jekyll blog.

## Thoughts

I've been on Blogger (the company Google bought to make blogspot.com) for 10 years now. I liked it because I don't write that much and it's got an Android app and Google's SSO/auth that makes it really easy to use. But it's also a simple WYSIWYG platform (at least when using the mobile app). Blogger has built-in Google Analytics, which is nice, too (I don't intend to sell any of my posts but it's nice to know that taking the time to write well is seen by others).

But Blogger has some short comings for me. Primarily: I want to write in markdown. I don't have to think much about how things are going to be displayed if I'm writing markdown and can just write with the simple syntax every so often and things look ok. Some of the other things I wanted included:

* More control over or being able to improve the theme
* More selection of themes to choose from
* Adding a menu
* Creating quoted text
* Tracking changes
* A nicer look on mobile devices

I think this last one made the quality of the content suffer quite a bit as there was no differenciation between what I was writing and the examples I was showing. Lastly, I wanted to be able to expand the platform - add a tab for pictures or automatically include a screenshot of a site I'm linking to or... I don't really know. But being able to alter the workflow is often fun.

## Where to?

I've used Hugo [0] for other Github Pages sites I've created. It works really well. However, it has some issues for me: I don't know golang [1] - the language Hugo is written in - and I don't want to learn it in order to hack on a framework that's building a site. I also enjoy working on mobile operating systems - namely Android - and Go apps need to be compiled for those ARM processors. Hugo is, however, a great platform - it works as advertised.

Instead, I went with Jekyll for this project. It's written in Ruby, which I know/have used before, and has a runtime compiled for linux running on Android. Ruby also has a fairly large community who use it for web sites (mainly due to Ruby on Rails), which I figure should have some bleed in for Jekyll. Jekyll is also written by Github, which has free web hosting via Github Pages. The template system for Jekyll is Liquid [2], which has some differences from Jinja2 or Erubis or Go templates that I've used before. I'm not thrilled about needing to know the differences with a template system I'm unlikely to use elsewhere, but the differences seem minor, and most of the time I'm in VIM writing markdown for Jekyll or not worrying about the template engine anyway. Last, I also wanted an excuse to use Jekyll instead of Hugo.

Using either Jekyll or Hugo SPA (single page application) site builder in my blog pipeline while using git and vim, makes me feel at home. This pipeline comes with some features Blogger can't provide. Vim is my editor of choice and has a great undo system that can show me what I was doing at any point in time. But I generally don't rely on that too much - what I do rely on is git's history. Knowing that I have a log that says "this blog is done being written" or "this blog has been proofread and is about to get published" and then being able to see what changed and move backward and forward in time is great. And lastly, git gives me about the best non-repudiation system I know about: gpg signed commits. If you ever question whether I wrote something, you can look at the signature in the Github repo for this site and verify I made the change (or at least used my hardware token to sign the change) [3].

## Shortcomings

A while ago, I bought a domain from Google - ioswitch.dev - it's one of many, but named well enough to put a Google Workspace on and use for professional email. I wanted to use the same domain I use for email for this blog. I definitely did not want to deal with Google/Postini domainkey migration if I didn't have to - if you've run an email server, you know you want to be as far away from them as possible. Changing Google Workspace settings was my absolute last choice.

The Google Domains web portal didn't let me add a CNAME record for the bare ioswitch.dev domain, but they do allow A records on the bare domain. I'm not sure if this has to do with how Google Workspace is setup - it seems odd, but again, I didn't want to change anything to do with how Workspace was setup, so I moved on. Well almost....

When I started this, I was hoping to use Amazon S3 and Cloudfront to host this blog. I could write a git subcommand [4] to push my work and then run jekyll to build the site and then run terraform to create and manage the infrastructure and upload the blog site. However, after a day of going down this path - with basically no one actually using the services I was setting up - I was already being charged $1. Does that mean my blog site was going to cost me $30 per month (or more)? I don't know. And there was still the issue of not being able to use a CNAME for my bare domain and AWS charging for public IP use (as does everyone else). I was looking at using Route53 to handle the domain and point that to an S3 bucket, which would've solved the external IP cost. But then I'd still need to move Google's domainkey to Route53 and deal with any issues that arise from that. As noted earlier, I did not want to deal with email issues.

## Chirpy

When you start configuring Jekyll (or Hugo), one of the first decisions you'll have to make is what theme to use. There are lists of themes - a list for Hugo [5] and one for Jekyll [6] - most seem based on themes for Wordpress. You'll look through them, click through the example sites they've setup, choose one, and then walk through the setup guide for whichever theme you decide to use. Pretty simple, but quite time consuming.

The Chirpy [7] project is a well done Jekyll theme that makes a clean blog site. It also comes with a Github Action that builds and tests the site [8]. This means that I don't need to write a function to run jekyll and terraform and deploy my blog. I would've preferred to handle deployment myself, but since the other way was going to cost money (this setup is completely free), I'll take the path of least resistance. Using the Chirpy Github Action also means that anyone can see when I update the site and what issues it has [9].

This means that the steps to make this all work are pretty simple: 

* I renamed the repo to `<username>.github.io`,
* Created the A records with the IPs that Github provides,
* Created a `www` subdomain with a CNAME pointing to `ag4ve.github.io`,
* Added the Github Action to my repo, and
* Told github about the custom domain.

And it just worked. The rest of these steps are well documented, but here's what my Google Domains dashboard looks like:

![Google Domains DNS dashboard for ioswitch.dev](/static/img/google_domains_ioswitch_settings.jpg)

I started with a blog repo based on Hugo and not the chirpy-starter repo. One of the things I did to convert this repo to use Chirpy is to add chirpy-starter repo as a remote and have been checking out files from that, as needed. Eventually I'm going to want to pull in updates from Chirpy. I'm unsure how I want to pull in Chirpy updates. Sense I've moved the blog, upstream changes to the Github Action that deploy the site have already broken and I had to checkout an upstream change (which isn't the best user experience). I've also noticed that when I link to a post on some social media sites, the header picture doesn't always show up - I'm going to need to fix that. Also, when I have notes at the top of a post, this text is picked up by those sites and that text appears out of context without the context of the rest of the blog post. I need to move these notes and ensure that doesn't happen again (with unit test). There are other technical issues with the site I need to fix like contact links not working correctly. None of this would be an issue on Blogspot, but I think the pros outweigh the cons, and I will fix these issues over time.

## Enhancements

So that people may one day stumble on this site through a web search, I've done a few things. I verified the site with Google here [10]. Then used their `PageSpeed Insights` under `Core Web Vitals` to generate reports [11] and [12], which I intend to work through over time. To give me an incentive to keep writing, I went to the Google Analytics page [13] and created a token for this web site and selected to enable it manually, then copied just the token from the JS code snip they provided and verified it works. Discus provides a comment section from Github discussions. While I originally didn't like the idea of people needing to sign in to comment, this is a technical blog hosted on Github, and having a comments section based on Github seems best. For this reason, I enabled Discus (see [14] for links and settings). Last, I converted my resume to markdown and made it my [About](/about) page.

## Looking back

Like everyone else, I use the internet. When looking for information, there are two things that annoy me more than anything else: a search engine showing results for pages that don't contain words I've quoted and broken links. I don't have any control over the former, but won't contribute to the later. Hence, everything on my old blog is going to remain as it is until someone (Google) decides to take it down.

I've transferred most of my old posts to the Jekyl ioswitch.dev website. The old content that I've changed is as follows. There's a blog on trading cryptocurrency that was amateurish investment advice - even though it worked for me at the time, it's not wise and not technical, and has no place here, but it's still in git history even if Blogspot goes away. Second, I wrote about having a computer as a phone a decade ago, thinking that I could turn a Raspberry Pi with a battery and screen into a phone with computer capabilities - I could and almost did, but other things have made that moot. That article is still here but not listed with everything else. Lastly, the Password Schemas blog gets some notes on not relying on it - I wrote that before I worked for a company that specialized in password audits - the math is acurate, but the theory isn't - so don't base your password strategy on what's there.

To close: everything that is currently at [https://ag4ve.blogspot.com](https://ag4ve.blogspot.com) will remain there - I have no plans to touch it. Moving forward, all new content will be at [https://ioswitch.dev](https://ioswitch.dev). As I'm not touching my old blog, it will not recieve any new content either. I hope you enjoy using my new blog platform.

[0]: https://gohugo.io
[1]: https://go.dev
[2]: https://shopify.github.io/liquid/
[3]: https://github.com/ag4ve/ag4ve.github.io/commits/master/
[4]: /posts/Getting-Git/#subcommands
[5]: https://themes.gohugo.io
[6]: http://jekyllthemes.org
[7]: https://github.com/cotes2020/jekyll-theme-chirpy
[8]: https://github.com/cotes2020/chirpy-starter/blob/main/.github/workflows/pages-deploy.yml
[9]: https://github.com/ag4ve/ag4ve.github.io/actions
[10]: https://search.google.com/search-console
[11]: /static/docs/Google_PageSpeed_Insights.pdf
[12]: /static/docs/Google_PageSpeed_Insights_mobile.pdf
[13]: https://analytics.google.com
[14]: https://github.com/ag4ve/ag4ve.github.io/blob/master/_config.yml#L72

