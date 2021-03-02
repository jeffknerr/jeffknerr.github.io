---
layout: post
title:  "Using multimap module in Rspamd"
date:   2021-03-02 14:12:35 -0500
categories: rspamd regex multimap
---

I wanted to use the multimap module in rspamd to block specific email
addresses and domains, as well as allow certain email addresses, ip
addresses, and domains.

Specifically, I want to make sure all emails from *myschool.edu* always
get through, and no emails from certain top-level domains ever get through
(e.g., `*.guru`, `*.info`).

The multimap module documentation has a lot of information, but doesn't
have examples of exactly what I wanted. It took me a few tries to get it 
working, and it's sometimes hard to test these if you are waiting for
the next email to trigger one of your maps, so I'm writing this up to help
others and also remember what I did.

NOTE: this is all running on a debian buster server running postfix 3.4
and rspamd 2.7 (March 2021).

## Example: block emails from whole domains

Here's the relevant part of the config section,
and the map file it uses, to block whole domains:

```
/etc/rspamd/local.d# cat multimap.conf
...
# raise score of emails from these domains
DENY_SENDER_DOMAIN {
      type = "from";
      filter = "email:domain";
      map = "/var/lib/rspamd/denysenderdomain.map";
      score = 20.0;
      regexp = true;
}
```

```
/var/lib/rspamd# cat denysenderdomain.map
candlewick.club
dearestpenst.shop
DEFENSEMECHANISM.INFO
/.*\.cyou/
/.*\.work/
/.*\.cam/
/.*\.buzz/
/.*\.top/
/.*\.info/
/.*\..*\.cam/
/.*\..*\.work/
/.*\..*\.cyou/
nedbank.co.za
redchillideals.co.za
chi.v3lomail.com
```

The above should add 20 points to the spam score for specific email
domains (like `dearestpenst.shop`), as well as any that match the
regular expressions. I'm not sure if the best way is to add points
to the score or just reject the email. Some of the configs below will
accept or reject the emails.

In the above regular expressions I am trying to
match "From" email domains such as `makdda.work` and
`mail.makdda.work`.

See the perldoc link for more info on regular expressions. Here's 
what the `/.*\..*\.cyou/` expression is trying to match:

    .* matches any single character (except newline) plus 0 or more of the preceding element (the dot element)
    \. matches an actual dot character
    .* another "any character plus 0 or more of any character"
    \. another actual dot
    cyou/   match should end with "cyou"

So that should match "a.b.cyou" and "hello.dear.cyou" and anything
else of the form "whatever.moretext.cyou".

## Example: allow emails from whole domains

I also have a similar "ALLOW_SENDER_DOMAIN" config that just
uses a different map file, and the spam score is decreased by an amount.
I chose -6 after watching the rspamd web user interface for a bit, and
seeing that some emails from my own domains were getting higher scores 
than I wanted (e.g., look at the History tab, select the "no action"
emails, and then click on the Score column to sort them -- then you can
easily see which of your own emails have the highest scores).

In the map file for this one I just put things like "myschool.edu" to 
make sure all emails from my school get a reduced spam score.

## Example: deny/allow sender

Here are the configs from `/etc/rspamd/local.d/multimap.conf`
for allowing and denying either specific email addresses or 
email addresses that match a pattern:

```
allowlist_sender {
          type = "from";
          filter = "email";
          map = "/var/lib/rspamd/allowsender.map";
          prefilter = true;
          action = "accept";
          description = "local sender allow list";
          symbol = "ALLOWLIST_SENDER";
          regexp = true;
}
DENYLIST_SENDER {
          type = "from";
          filter = "email";
          map = "/var/lib/rspamd/denysender.map";
          prefilter = true;
          action = "reject";
          description = "local sender deny list";
          regexp = true;
          score = 20.0;
}
```

Again, I wasn't sure if the action or the score was working when I set this
up. Looking at the logs now, it appears any messages that match the 
entries in the `denysender.map`
file get a score of 20 and also get rejected, so that's probably redundant.
Any that match my `allowsender.map` entries get a score of 0 and get accepted
(without looking at other symbols or doing other checks).

In these map files I have both specific email addresses to deny (`jobs@target.com`)
and to allow (`root@myschool.edu`), as well as regular expressions like
`/.*@.*\.buzz/` and `/.*@myschool.edu/`.

## Example: deny/allow ip address

Regular expressions are not allowed for the `type = "ip"` attribute,
so these map files just have complete ipv4 addresses in them.

```
blocklist_ip {
          type = "ip";
          map = "/var/lib/rspamd/blockip.map";
          prefilter = true;
          action = "reject";
          description = "local block list";
          symbol = "BLOCKLIST_IP";
}
allowlist_ip {
          type = "ip";
          map = "/var/lib/rspamd/allowip.map";
          prefilter = true;
          action = "accept";
          description = "local allow list";
          symbol = "ALLOWLIST_IP";
}
```

And again, when I set this up, I didn't know about the score attribute,
so I just rejected or accepted the email if it matched the ip address.

These ip maps seem a little less useful, as spams using the same email
address or domain seem to come from many different ip addresses. Still,
might be useful if specific IP addresses are pounding you with spam.

## more info 

Check out the [Rspamd docs][rspamd] and the [multimap module][multimap] for more info.
If you need more help with regular expressions, see the [perldoc pages][perlinfo].

[rspamd]: https://rspamd.com/
[multimap]: https://rspamd.com/doc/modules/multimap.html
[perlinfo]: https://perldoc.perl.org/perlre

