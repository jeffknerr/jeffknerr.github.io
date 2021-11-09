---
layout: post
title:  "aws static site"
date:   2021-11-09 16:37:00 -0500
categories: aws static site
---

# everything I did to put up a static site on aws

I wanted to learn a little about aws, so I tried setting
up a static website. Below is everything I did (took me a few tries).
The [static test website is here][jmktest], if you are interested
(the content is for a project my wife is working on).
NOTE: I didn't put much effort into making the site look pretty. :(

I mostly used a [document I bought from Kyle Galbraith][kg] (I think
there was a half-off sale at one point). I bought it in 2019, but just
now pushed through most of it and set up a site. It was worth it for 
me, but I'm sure you can find other blogs and articles that are free
if you want.

I used [Jekyll][jekyll] for the static site.

## docs I followed

- [Jekyll Docs][jekyll]
- [Kyle Galbraith's *Learn AWS By Using It*][kg]
- [Marcos Lombog's *Build and Deploy a Jekyll Website*](https://betterprogramming.pub/build-a-static-website-with-jekyll-and-automatically-deploy-it-to-aws-s3-using-circle-ci-26c1b266e91f)
- [amazon s3 developer guide](https://github.com/awsdocs/amazon-s3-developer-guide/blob/master/doc_source/website-hosting-custom-domain-walkthrough.md)

## overview

* signed up for an aws account
* set up two-factor auth 
* set up second non-root IAM account
* set up two-factor auth for the IAM account
* enabled billing alerts, set an alarm ($1)
* installed awscli on my local workstation (running ubuntu)
* using Route53, bought a domain (jmktest.net) for $11/year
* using S3, made the www.jmktest.net bucket
* made bucket publicly readable
* configured the bucket for static website hosting
* set the properties for the bucket
* made a naked domain S3 bucket (jmktest.net)
* set this bucket to just redirect to www.jmktest.net bucket
* using Route53, set up DNS record for www to point to the S3 bucket
* set up my jekyll site on my local computer
* uploaded my `_site` directory to the S3 bucket
* tried out the site (not using https yet) to make sure it worked
* made a CloudFront Web distribution for the S3 bucket
* updated my DNS to point to this new CloudFront distribution
* used AWS Certificate Manager to request/create an SSL cert for www.jmktest.net,
including jmktest.net as another name
* added the SSL cert to the CloudFront distribution
* now tried the site using https

This all looks straightforward, and I'll show more details below, but
it took me many tries to get it all to work. The dns and caching make
it difficult to debug things quickly. I'm still not sure what I did 
to finally get it to work, which is why I am writing it all up here.
Ideally I would do this all again, using a new domain, and make sure
these steps all work...

## the details


## still to do

- autodeploy to aws when I git commit/push
- optimize the distribution for caching??


[jmktest]: https://www.jmktest.net
[kg]: https://www.kylegalbraith.com/learn-aws/
[jekyll]: https://jekyllrb.com/docs/
