---
layout: post
published: true
title: Low friction Octopress/GitHub pages Setup
comments: true
---

My blog uses [Octopress](http://octopress.org) which is basically Jekyll plus a whole bunch of plug-ins preconfigured and it is hosted on [GitHub Pages](https://pages.github.com). But sometimes I really miss being able to simply create or edit posts online. I started looking around and found [Prose.io](http://prose.io/).

Prose is an *awesome* open source online Jekyll site editor for GitHub. Then if we setup TeamCity to automatically regenerate and deploy our blog when we make any commits we have a really simple way of making quick blog posts online.
<!-- more -->
Here is a quick rundown of creating a new post with Prose. If you are only interested in the automated TeamCity deployment just skip this section.

1. Go into the _posts folder, then click New File  
![New file](/assets/posts/Prose1.png)
2. Add a title and write your post  
![Write post](/assets/posts/Prose2.png)
3. You can define your yaml metadata defaults for prose in your _config.yaml, giving you a nice UI  
![Edit metadata](/assets/posts/Prose3.png)

When you hit save Prose will commit that file, pretty sweet. **note** if you do not create the file in the _posts folder you will not see the meta-data options. **note2** Image uploading is broken at the moment, hopefully this gets some love soon

### My Prose Config
You can configure Prose by adding some additional metadata in your _config.yaml file. Here is mine:

    # ----------------------- #
    #   prose.io settings     #
    # ----------------------- #
    prose:
      rooturl: "source"
      site: "http://jake.ginnivan.net"
      media: "source/assets/posts"
      metadata:
        "source/_posts":
          - name: "layout"
            field:
              element: "hidden"
              value: "post"
          - name: "title"
            field:
              element: "text"
              value: "Title"
          - name: "comments"
            field:
              label: "Allow comments"
              element: "checkbox"
              value: true
          - name: "categories"
            field:
              element: "text"
              value: ""
          - name: "published"
            field:
              label: "Published"
              element: "checkbox"
              value: true
              
Read more about [Prose Configuration](https://github.com/prose/prose/wiki/Prose-Configuration).

## Setting up continuous deployment from TeamCity
The next step is I want TeamCity to automatically regenerate my Octopress site and push it to GitHub. This also assumes you have your site setup as a **user** site

1. Install everything you would need to get Octopress building locally on your build server
1. Create new build
1. Add VCS root pointing at `source`
  1. Set TeamCity Checkout mode to *checkout on agent*
1. Add `Command Line` Build Step with Exe of `bundle` and Parameter of `install`. This will install all the dependencies Octopress needs
1. Next we need TeamCity to clone our repo into the `_deploy` directory, we can do this with another Command Line build step with the *Run* parameter set to Custom Script (the reason for `git checkout master` is that I have the source branch set as default, and the _deploy branch needs to be on master)

        if not exist _deploy (git clone https://%GithubUsername%:%GithubPassword%@github.com/%GithubUsername%/%GithubUsername%.github.io.git _deploy)
        cd _deploy
        git checkout master
        cd ..

1. The last build step is to regenerate and deploy our Octopress site. Add a `Rake` build step with the path set to `Rakefile` and the task set to `gen_deploy`. Make sure the *bundle exec* option is checked
1. Now setup a VCS build trigger so we deploy whenever anything changes
1. Finally go to the *Parameters* section of your build configuration. There will be two parameters needing values, *GitHubUsername* and *GitHubPassword*. *GitHubUsername* is easy, just put your username in. 
  - For *GitHubPassword* click on `Edit` next to Spec, then set Display to Hidden and Type to Password. This will make sure your password does not show up in build logs and will be \*\*\*\*'d out instead.

Save your build configuration, and now whenever you push any updates to `source` your blog will be regenerated and deployed.

Enjoy!