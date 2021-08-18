---
layout: post
title:  "asciidoc static site on github pages"
date:   2021-06-15 14:40:00 -0500
categories: asciidoctor github python reference
---
I wrote a [python reference doc][pyref] for our intro Comp Sci students, 
and I wanted to host the repo here on github, but also host the
static website, which is written in [asciidoc](https://asciidoc.org/).

This post shows the steps needed (one way) to turn asciidoc files
into a static website as part of your github pages (a separate repo that gets
published to a sub-url of your github pages).

## docs I followed

- [github docs on setting up a "continuous integration workflow"][workflow]
- [Manoel Campos da Silva Filho's AsciiDoctor GitHub Pages Action][manoel]
- [Owais Ali's post on Hosting a Static Website on GitHub][owais]

And the final result is this [python reference doc][pyref].


## the setup

* make a new github repo
* clone it
* add some adoc files
* note: I think Manoel's github action needs a README.adoc file, so make sure you at least have that
* git commit/push the adoc files back to github
* following the [github workflow doc][workflow]:
  - click on "Actions" (on main repo page)
  - click "Set up this workflow" for the simple "Blank" workflow file
  - "Start commit"
  - Commit directly to the "main" branch
  - "Propose new file"
* now back in your repo, run a "git pull" and you should see a new `.github/workflow/blank.yml` file
* edit the `blank.yml` file to add [Manoel's][manoel] `asciidoctor-ghpages` addition:
```
    # Includes the AsciiDoctor GitHub Pages Action to convert adoc files to html and publish to gh-pages branch
    - name: asciidoctor-ghpages
      uses: manoelcampos/asciidoctor-ghpages-action@v2
      with:
        asciidoctor_params: --attribute=nofooter
```
* git commit/push that back to github
* now back on your repo's github.com page:
  - click "Settings"
  - go to/find the GitHub Pages tab (just above the Danger Zone)
  - under Source, select the `gh-pages` branch
  - then click `Save`

 I think at this point you need one more git push to activate and build
 the whole thing, so make another edit in one of your adoc files and
 git commit/push the change.

 After a git push, if you go to your github repo Actions tab, you should
 now see the running workflow, and hopefully a green check mark. If it's 
 not green, you can click on the workflow to see any debugging info.

 If everything worked your site should be deployed to https://USERNAME.github.io/REPONAME




[pyref]: https://jeffknerr.github.io/pythonreference/
[manoel]: https://github.com/manoelcampos/asciidoctor-ghpages-action
[workflow]: https://docs.github.com/en/actions/guides/setting-up-continuous-integration-using-workflow-templates
[owais]: https://medium.com/any-writers/how-to-host-a-static-website-on-github-for-free-f47b12790775
