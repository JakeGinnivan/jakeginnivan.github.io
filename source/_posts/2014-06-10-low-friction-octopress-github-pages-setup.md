---
layout: post
published: false
title: Low friction Octopress/GitHub pages Setup
comments: true
---

## Low friction Octopress/GitHub pages Setup

My blog uses [Octopress](http://octopress.org) which is basically Jekyll plus a whole bunch of plug-ins preconfigured and it is hosted on [GitHub Pages](https://pages.github.com). But sometimes I really miss being able to simply create or edit posts online. I started looking around and found [Prose](http://prose.io/).

Prose is an *awesome* open source online Jekyll site editor for GitHub. Here is a quick rundown of creating a new post.

1. Go into the _posts folder, then click New File  
![New file](/source/assets/posts/Prose1.PNG)
2. Add a title and write your post  
![Write post](/source/assets/posts/octopress-setup/Prose2.PNG)
3. You can define your yaml metadata defaults for prose in your _config.yaml, giving you a nice UI  
![Edit metadata](/source/assets/posts/octopress-setup/Prose3.PNG)

When you hit save Prose will commit that file, pretty sweet. **note** if you do not create the file in the _posts folder you will not see the meta-data options.

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

