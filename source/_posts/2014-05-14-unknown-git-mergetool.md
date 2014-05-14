---
layout: post
title: "Unknown git mergetool"
date: 2014-05-14 20:08:32 +0100
comments: true
categories: 
---

Today my git mergetool stopped working. When I ran `git mergetool` I was greeted with:

	git config option merge.tool set to unknown tool: --global
	Resetting to default...
	 
	This message is displayed because 'merge.tool' is not configured.
	See 'git mergetool --tool-help' or 'git help config' for more details.
	'git mergetool' will now attempt to use one of the following tools:
	tortoisemerge emerge vimdiff
	No known merge tool is available.

Somehow (seriously, I have **no** idea how I did this) I had created a setting in one of my repositories set my mergetool to `--global`.

If you happen to get yourself into the same issue, run `git config --list` to see what your config settings are.

For me, I had a rogue `merge.tool` entry, so I just had to run `git config --unset merge.tool` which deleted the entry and I was off again.