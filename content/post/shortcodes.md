---
title: "Fixing the Hugo Twitter Shortcode"
date: 2018-12-07T20:28:17+01:00
draft: false
excerpt: Shortcodes in Hugo come in handy when you don't want to mess with a lot of HTML boilerplate. Sometimes though, you might want a little different behavior than what the default shortcodes provide. Here's a quick fix for the Twitter shortcode to suppress the thread view.
tags:
- hugo
---

Shortcodes in my favorite blog engine [Hugo](https://gohugo.io/) come in handy when you don't want to mess with a lot of HTML boilerplate. There are times though, where you might want a little different behavior than what the default shortcodes provide. Here's a quick fix for the Twitter shortcode to suppress the default thread view.

In my posts I often use the `{{</* tweet */>}}` shortcode for adding Twitter tweets. This works in 99% of the cases. However, sometimes I want to reference an answer in a Twitter thread, for example this one from my colleague [Alex Ellis](@alexellisuk), the founder of the [OpenFaaS](https://docs.openfaas.com/) project.

<center>{{< tweet 1069487297505185792>}}</center>

You can see in the example that it also shows the parent tweet. This is the default for the Twitter shortcode implementation compiled into Hugo. In my case, I often don't want to include the history of a tweet. Turns out it's pretty easy to fix in Hugo with [custom shortcodes](https://gohugo.io/content-management/shortcodes/).

First, enter the root of your Hugo blog folder and create an empty file for our custom Twitter shortcode. The name of the file is important as it also defines how you call it in your blog post files (`*.md`).

```bash
$ cd <HUGO_BLOG_ROOT>
$ touch layouts/shortcodes/tweet-single.html
```

Enter the following snippet in the `tweet-single.html` file. 

```json
{{- $pc := .Page.Site.Config.Privacy.Twitter -}}
{{- if not $pc.Disable -}}
{{- if $pc.Simple -}}
{{ template "_internal/shortcodes/twitter_simple.html" . }}
{{- else -}}
{{- $url := printf "https://api.twitter.com/1/statuses/oembed.json?hide_thread=1&id=%s&dnt=%t" (index .Params 0) $pc.EnableDNT -}}
{{- $json := getJSON $url -}}
{{ $json.html | safeHTML }}
{{- end -}}
{{- end -}}
```

This code is copied from the Hugo source as of commit `4b5f743`. Make sure you constantly reflect important changes to the upstream implementation, e.g. security/privacy related. The only parameter we had to add, `hide_thread=1`, tells the Twitter API to only respond with the single tweet referenced by the dynamically generated `id` parameter when Hugo renders your site. 

**Note:**  This custom shortcode falls back to the default implementation if you use `Privacy.Twitter=simple` in your Hugo configuration.

After we save the file let's give it a spin with our custom shortcode. This is how you would use it: `{{</* tweet-single 1069487297505185792 */>}}`

<center>{{< tweet-single 1069487297505185792>}}</center>

It works! And the best: Since we did not overwrite/replace the default shortcode, you can switch between them as needed!