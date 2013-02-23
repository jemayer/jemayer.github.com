# HOWTO (GIT Workflow)

Also see: http://blog.zerosharp.com/clone-your-octopress-to-blog-from-two-places/

## Cloning an existing Octopress blog to a new machine

First, clone the source branch to the local octopress folder:

    $ git clone -b source git@github.com:jemayer/jemayer.github.com.git octopress

Then, clone the master branch to the _deploy subfolder:

    $ cd octopress
    $ git clone git@github.com:jemayer/jemayer.github.com.git _deploy

Run the rake installation to configure everything:

    $ gem install bundler
    $ bundle install
    $ rake setup_github_pages

That’s you setup with a new local copy of your Octopress blog.

## Pushing & pulling changes from two different machines

Make sure to push everything before switching computers.

From the first machine, do the following whenever changes were made:

    $ rake generate
    $ git add .
    $ git commit -am "Some comment here."
    $ git push origin source  # update the remote source branch
    $ rake deploy             # update the remote master branch

Then on the other machine, you need to pull those changes:

    $ cd github-octopress
    $ git pull origin source  # update the local source branch
    $ cd ./_deploy
    $ git pull origin master  # update the local master branch

## In general: Publish posts and pages, backup source

After the usual `rake new_post['foo']` or `rake new_page['bar']` procedure, generate and publish new cotent:

    $ rake generate
    $ rake deploy

Commit corresponding sources:

    $ git add .
    $ git commit -m 'Yet another meaningful commit message'
    $ git push origin source

Be aware that generated static blog content lives on the master branch (`_deploy`), representing the web root directory on GitHub. Blog sources belong to the source branch.
