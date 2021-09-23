---
title: "Creating a dynamic review system for a static Hugo site"
date: 2021-09-23T12:58:17+01:00
image: uploads/blog-post-01.jpg
feature_image: uploads/blog-details-image.jpg
author: Alex Williams
---

> Using a Golang backend, we can create a rich form system that sends emails,
> raises pull requests, and does some basic spam protection.

<!--more-->

Static website generators are great. However, the biggest drawback is their also
their biggest strength, they are static. Often we need forms to collect
information from our users, most common is the trusty contact form. Things like
[formspree.io] are good for this but require a subscription. This is totally
fine for a commercial website as the plans are often affordable, but for hobby
sites it isn't ideal. Similarly, dynamic content we can use [Staticman] which
raises pull requests with new content. The work I did here is heavily inspired
by [Staticman].

I, like most developers that make websites as a hobby, completely over engineer
my sites and use them as a playground to learn new things. For example I deploy
my sites to a Kubernetes cluster in digitalocean, because I can not because I
should. Which means that running a small `golang` server to act as a form
backend, and also host the statically generated html wouldn't be too hard.

The website that I was working on, [High Heath Farm Cattery], has 3 forms:

- a simple contact form,
- a booking form registering interest in a stay for you cat,
- and a review form, for letting users share their experience of using the
  cattery



[formspree.io]: https://formspree.io/
[Staticman]: https://staticman.net/
[High Heath Farm Cattery]: https://www.highheathcattery.co.uk/
