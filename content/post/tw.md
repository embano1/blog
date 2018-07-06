---
title: "When Twitter Likes become your Bookmarks"
date: 2018-07-05T21:10:15+02:00
draft: false
excerpt: "If you're like me, you might (mis)use Twitter's Like button as a way to bookmark important tweets. Once you've reached several hundred or thousands of bookmarks, ehm Likes, it becomes really hard to find what you're looking for. Mainly because Twitter Likes are not designed for this type of use and the twitter.com UI search is not super flexible. However, with some CLI magic, we can easily fix that :)"
tags:
- Twitter
- JSON
---

<img src="https://cdn.vox-cdn.com/thumbor/kA-ikIyInItp2OuWG_dd1PfrFo4=/0x16:2200x1483/1820x1213/filters:focal(0x16:2200x1483):format(webp)/cdn.vox-cdn.com/uploads/chorus_image/image/47574011/twitter-hearts-and-stars.0.0.png" width="80" title="Source: The Verge"></img>

I confess, I've been (mis)using Twitter's famous *Like* button since joining this great platform. Thus, if you look at my <a href="https://twitter.com/embano1/likes" target="_blank">Twitter Likes</a> you'll mainly find tweets about distributed systems, Go and Kubernetes.

Yes, I know it's wrong! But looks like I am <a href="https://twitter.com/mkaptano/status/948588218009649152" target="_blank">not</a> the <a href="https://twitter.com/JamesMunnelly/status/1014982229989183493" target="_blank">only one</a> going down this dark path :) Unfortunately, if you have hundreds or thousands of bookmarks, ehm *Likes*, finding the one you desperately need is almost impossible due to the way the twitter.com UI is designed.

I am not using any native Twitter client on OSX. In case your favorite client (Tweetbot?) already provides that functionality for you, including regular expression support and querying advanced Twitter API fields, nice! You might want to stop reading here. But in case you're interested in a way to do that programmatically on the command line (CLI) of your operating system of choice with full flexibility and JSON support, read on. You might also learn some neat JSON filters along the way :)

Typically, I need to perform these actions when searching through my Twitter "bookmarks":

- Filter tweets for a **specific string**, e.g. `"Kubernetes"`
- Look for tweets **containing "A" and "B"**; order and capitals should not matter, e.g. `"...best practices...Golang"` or `"Go...Best Practices"`
- Filter tweets containing **only Youtube videos** of a specific topic or from a specific user; this requires access to some extended Twitter API fields because links in tweets are typically shortened to `https://t.co/...`

This and much more can be done with the powerful combination of two CLI tools: `tw` and `jq`. 

<a href="https://github.com/embano1/tw" target="_blank">tw</a> is a tool I created to easily access the Twitter API from your CLI. At the time of writing this post, only *Likes* are supported. <a href="https://stedolan.github.io/jq/" target="_blank">jq</a> is a *lightweight and flexible command-line JSON processor*. 

Of course, you can also use `grep` or similar tools to just filter for a specific string. `tw` has a `--pretty` option which generates a compact summary of your Twitter *Likes* (Author, Text, Link) which makes filtering with `grep` easy. For advanced queries, `tw`'s JSON output from the Twitter API and a processor like `jq` is the way to go. Scott S. Lowe provides some alternatives to `jq` in <a href="https://blog.scottlowe.org/2018/06/28/more-handy-cli-tools-json/" target="_blank">this post</a>.

## Getting the tools

`tw` is a statically compiled Go binary which you can easily install on Windows, Linux and OSX. You can either grab a <a href="https://github.com/embano1/tw/releases" target="_blank">release</a>, use *Homebrew* to install it on OSX, use Docker or build from source (see <a href="https://github.com/embano1/tw#get-it" target="_blank">README</a> in the Github repository).

With Homebrew: `brew install embano1/tw/tw`  
With Docker: `docker pull embano1/tw`

The Docker image contains `jq`. If you don't want to use the Docker image, please install `jq` as well to follow along with the examples. You can grab a release <a href="https://stedolan.github.io/jq/" target="_blank">here</a> or, e.g. for OSX, install via Homebrew: `brew install jq`

## Accessing the Twitter API

In order to use Twitter's API one has to authenticate. This is to protect the Twitter service.

> Twitter uses OAuth to provide authorized access to its API.

As such, `tw` needs credentials before it can make requests against the Twitter API. A step-by-step guide is provided in the <a href="https://github.com/embano1/tw#authentication" target="_blank">README section</a>. Please make sure to set this up before proceeding.

(Waiting...)

Ok, welcome back! Let's show the power of both tools in action. 

## Examples

We'll use the Docker image for our examples. The examples assume that you stored your Twitter API credentials in the file `~/auth.json`. 

Let's test if everything is working.

```bash
# Start interactive session and shell
$ docker run -it -v ~/auth.json:/auth.json embano1/tw sh

# Query your likes using pretty printing (-p) 
/ tw -f auth.json likes -p
(...)
-----------------------
From: golangweekly
Text: "How I Structure Production Grade REST APIs in Go: https://t.co/UGf25MuV9P (The initial post in the series, it focuses on application structure and routing.)"
Link: https://medium.com/@tonyalaribe/structuring-a-production-grade-rest-api-in-golang-c0229b3feedc
-----------------------
(...)
```

Thumbs up!

If `tw` complains about missing/incorrect credentials, make sure you correctly followed the section "Accessing the Twitter API" in this post. Depending on the number of *Likes* it might take a while to query the API.

**Note:** The Twitter API uses rate limiting to protect against (D)DoS attacks. To prevent throttling or timeouts, I recommend dumping your Tweets into a file first and use that for filtering. It's also much faster, especially when querying thousands of *Likes*.

Let's move on with some concrete examples I use every day. We continue to use our Docker container session which started for our test above.

### #1 Only print Tweets containing "String" (ignore case)

First, dump all Tweets into a file.

```bash
/ tw -f auth.json likes > tweets.json
```

Since we want to do a *case-insensitive search*, we cannot use the `contains` function in `jq`. Fortunately, `jq` supports regular expressions, *regex* for short, with the `match` function. 

```bash
/ jq '.[]|select(.full_text|match("handy";"i"))|.full_text' tweets.json
(...)
"[New Post] More Handy CLI Tools for JSON: https://t.co/CxAWaJbfI5"
"\"Choosing an HTTP Status Code\"\n\nA handy flowchart for figuring out which one applies in your situation.\n\n(Sadly no paths lead to 418...  )\n\nhttps://t.co/utMDLotwJx"
(...)
```

Breaking down this command:

- `.[]` # query all array members in the JSON file
- `|` # pipe the left object(s) to the next command, in our case the whole array
- `select(.full_text|match` # use object field `full_text` and pipe it to the regex filter (`match`)
- `match("handy";"i")` # filter for the string `"handy"` (would also match the word "unhandy" for example), ignore case with `;"i"`
- `|.full_text` # pipe the result to next command; if there is a match, only output the `full_text` field

If you want to search for the exact word, e.g. excluding "unhandy" from results:

```bash
/ jq '.[]|select(.full_text|match("\\b(handy)\\b";"i"))|.full_text' tweets.json
```

### #2 Only find Tweets with these "two" "Words" (ignore case and order)

Sometimes you want to search for two (or more) words, but don't know the order. For example, I typically search for "best practices" in combination with a technology, e.g. "Docker" or "Kubernetes". Regular expressions with *lookahead* conditions are what we need here.

Let's filter for "Go" respectively "Golang" related tweets also mentioning "production", "practice" or "idiomatic". Again, this regex would also match "practices" since we don't ask for exact word matching here (not using `\b` word boundary checker in regex).

```bash
$ jq '.[]| select( .full_text | match("(?=.*(\\bGo\\b|golang))(?=.*(production|practice|idiomatic))";"i"))|.text' tweets.json
(...)
"How I Structure Production Grade REST APIs in Go: https://t.co/UGf25MuV9P (The initial post in the series, it focuses on application structure and routing.)"
"Building Scalable Web Services in Go: https://t.co/2Ul9ibqopd (A few best practices aggregated from around the world of Go.)"
"List of articles discussing \"Idiomatic Go\"\n\nhttps://t.co/LWUvxxmTCc\n\n#golang"
(...)
```

Breaking down this command:

- `.[]| select( .full_text |` # we want to apply our filter to `full_text` entries from our tweets
- `match()` # we want regex
- `"(?=.*(\\bGo\\b|golang))(?=.*(production|practic|idiomatic))"` # *lookahead* syntax for regex; simply speaking order of our two filters (something `Go OR Golang` and `production OR practice OR idiomatic`) does not matter
- Note that the `|` in regex queries means *logic OR* and **not** pipe to next filter 
- `;"i"))|.text` # case-insensive regex; if there is a match, only print tweet `text`

### #3 Only print "Field(s)" we're interested in

Sometimes all you want is a simple Tweet representation, e.g. by *User* (incl. full name), *Tweet* and *Link(s)*, if any. If `tw`'s `--pretty` option is not enough, using `jq` arrays and maps is the right approach. It also keeps the JSON format, so you can do further filtering on the array/map.

```bash
/ jq '[.[]|{user: .user.screen_name,name: .user.name, text: .full_text, url: [.entities.urls[].expanded_url]}]' tweets.json
(...)
[
  {
    "user": "timoreimann",
    "name": "Timo Reimann",
    "text": "@the_sttts @TheNikhita https://t.co/XVdwwzJxXX is comprehensive, though it might be a bit overwhelming depending on how much you know already.\n\nGoogle's style guide does a good job to explain bash behav
ior: https://t.co/z4sfr0oiKj\n\nFinally, enabling shellcheck is a great way to learn while scripting.",
    "url": [
      "https://mywiki.wooledge.org/BashFAQ",
      "https://google.github.io/styleguide/shell.xml"
    ]
  },
  {
    "user": "vCabbage",
    "name": "Kale Blankenship",
    "text": "@copyconstruct https://t.co/lziacf4S4q works pretty well for this.",
    "url": [
      "https://github.com/fortytw2/leaktest"
    ]
  },
(...)
```

Breaking down this command:

- Outermost `[...]` # create one resulting array for the output of the map/filter specified
- `.[]|{user: .user.screen_name,...` # for each input object create a new map mapping the specified input fields to custom ones (left side of `:`)
- `url: [.entities.urls[].expanded_url]}` # `.urls[]` could be empty, one or multiple, thus creating an array (with `[]`) for the mapped entry `url:`

### #4 Filter for Tweets with Links to Youtube videos

There's always an amazing Go or distributed systems talk which I bookmark to watch later, e.g. on these long flights to the US. Filtering for Youtube tweets is not that easy without JSON because Twitter shortens links (`t.co...`). 

Therefore we need to access an extended JSON field from our tweets (`.entities.urls[].expanded_url`). 

```bash
$ jq '[.[]|select( .entities.urls[].expanded_url | contains ("youtube"))|{text: .full_text, url: [.entities.urls[].expanded_url]}]' tweets.json
(...)
  {
    "text": "\"Google Production Environment\"\n\nAn intro to the infra that allows running processes reliably and scalably in data centers, as well as the development and build infrastructure that enables engineers to develop, deploy, and run large-scale services.\n\nhttps://t.co/W8nkNJlfPM",
    "url": [
      "https://www.youtube.com/watch?v=dhTVVWzpc4Q"
    ]
  },
  {
    "text": "And my talk from yesterday about @kubernetes as an API driven platform – API concepts, CRDs and controllers – from our Reykjavík Kubernetes Meetup 🇮🇸 https://t.co/cLg5EFDDPD",
    "url": [
      "https://www.youtube.com/watch?v=BiE7oKeEzDU"
    ]
  },
(...)
```

We filter whether `.entities.urls[].expanded_url` contains the string `youtube` (case-sensitive, thus `contains` function works here) and pipe that into a map only containing the tweet text and url(s). The outermost brackets will create an array for us. So we could for example only print the first three items with a final filter: `|.[:3]` before closing the `jq` filter/query with `'`.

I hope you enjoyed this post. Yes, `jq` filters can be complex and hard to use, especially when not using them on a daily basis. For that reason, I maintain a Gist of filters I use regularly for `tw` <a href="https://gist.github.com/embano1/1631a6d44eaa7a934807ec80c7cc74e0" target="_blank">here</a>. 