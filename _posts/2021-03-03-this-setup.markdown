---
layout: post
title:  "this setup"
date:   2021-03-03 11:06:10 -0500
categories: jekyll markdown github pages
---
Here are my notes from making this site. I will
add to it as I learn more jekyll and how the
github.io stuff works.

# docs I followed

- [University of Toronto Coders: Web Design With Jekyll/Github Pages][utc] 
- [Jekyll Docs][jekyll]

## install jekyll on my computer

My home computer is currently (March 2021) running Ubuntu 20.04.
I followed the [Jekyll on Ubuntu](https://jekyllrb.com/docs/installation/ubuntu/)
page, except I use [zsh](https://ohmyz.sh/), 
so I edited my `.zshrc` file instead of the `.bashrc` file.

Prerequisites:

```
sudo apt-get install ruby-full build-essential zlib1g-dev
```

Zsh stuff:

```
$ grep gem ~/.zshrc
export GEM_HOME="$HOME/gems"
export PATH="$HOME/gems/bin:$PATH"
```

(and `source ~/.zshrc`).

Install jekyll and bundler:

```
gem install jekyll bundler
```

Note: I think this installs new "gems" into my ~/gems directory, so I can
install them without root.

## set up github.io repository

I already had an account on [github](https://github.com), so I just had
to add a new repo with the name `username.github.io`. See 
[https://pages.github.com/](https://pages.github.com/) for more info on this step.

## add stuff/deploy

Lastly, I want to work on this site/page locally, then deploy it all
back to github. I'm still figuring this part out a bit.

- clone the above *username.github.io* repo (which is still empty) to my local computer
- run the `jekyll new` command in the directory *above* that cloned repo (I had to use the `--force`??)

```
git clone git@github.com:username/username.github.io.git
jekyll new username.github.io --force
cd username.github.io
jekyll serve
```

At this point you should have a basic jekyll site running on `http://127.0.0.1:4000`.

- change the `about.markdown` file a bit
- change the `_config.yml` file a bit
- run `jekyll build` and `jekyll serve` if you want to see it locally again
- add new files to git and push it back

```
git status
git add .
git commit -a -m 'first try at jekyll and github pages'
git push
```

Now your site should be up at **https://username.github.io** (actually, it might
take a minute or two to finish the full deploy).

## things to fix

- I get a **running version of Bundler (2.1.2) is older than...** warning when I run `jekyll serve`
- find a better [jekyll theme](https://github.com/aterenin/minima-reboot)
- fix up the front page and all of the little things...


[utc]: https://uoftcoders.github.io/studyGroup/lessons/misc/jekyll-ghpages/lesson/
[jekyll]: https://jekyllrb.com/docs/

