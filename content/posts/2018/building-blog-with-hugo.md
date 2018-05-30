---
title: "Building a Blog With Hugo (Eventually)"
date: 2018-01-17T22:24:53+10:00
draft: false
tags: [hugo, blog]
categories: [Software, Exploration]
---

I have tried many times to start a blog. I think writing often is important for me
because writing is something I have difficulty concentrating on. Some
might find my distaste and associated boredom with writing ironic given I
elected to do a PhD... but I digress.

## Required features

These are the requirement I have always wanted for a blog:

- Free hosting
- Complete customisation
- Complete transparency--As a professional point of pride, 
  I need to understand how it all works
- Minimal overhead--I don't want to be patching (even more) servers

Putting those requirements together, I decided I wanted to use a static website
generator. Ever since GitHub Pages introduced me to the concept of static site
generators, I've found it ridiculous that WordPress required an active
installation of a web server and a database. All this extra overhead for running
a dynamic web application is just too much complexity for what a blog (or for
that matter many other websites) need.

I decided on these requirements years ago. If I had settled for 
a hosted blogging service I would have been up, running, and
happy years ago.

Nonetheless, unreasonable persistence.

## Attempt 1: Jekyll + GitHub Pages

Jekyll has been the mainstay for creating static blogs for a while. It's written in Ruby
and has mainly risen in popularity due to support from GitHub Pages--free building
and hosting when you commit to your repository.

However Jekyll has annoyed me:

- It is slow, painfully slow. For even a small blog I was seeing 16 second build
  times!
- Because it was written in Ruby it could be hard to set up on Windows.
- In more recent versions, more and more dependencies are introduced, many with
  native code dependencies (which makes it even harder to compile).
  Massive toolchains are the opposite of the simplicity I desired.
- And lastly, Jekyll can be surprisingly hard to restructure.

Things are rigid. If you veer out of your lane you get pushed back in. 
Jekyll has collections
that let you have multiple, well,  uh, "collections" of things, like bog posts. It also
supports static pages. However, it treats these concepts as separate entities:
you can't get the useful properties of one in another. There's no recursive
collection of static pages available as a variable, which makes arbitrary-depth
static page collections somehow infeasible.

To be honest I don't remember the problems clearly, but I do remember the frustration.

## Attempt 2: MetalSmith

Metalsmith is a Node bare-bones static site generator built using a middleware chain. 

In theory, all you had to is stitch a collection of middleware together. You
would include just the features you want, and if you wanted more it was as simple
as adding another function to the chain. Nearly everything you needed could be
added as a middleware layer, and if it wasn't
available it is easy to write your own middleware.

I thought it was a great idea. Simple enough to understand because you need to
compose it by yourself. Powerful enough that you could achieve
almost anything, and thus was customisable.

However, I found the level of abstraction to be profoundly frustrating. Early on
I wanted to do several things (like debugging, logging, or measuring performance)
which was just plain impossible because the middleware interface was too simple.
There were no hooks like `afterMiddleware` or `beforeMiddleware` and so the only
way to customise the pipeline was to wrap every middleware in my own handler, or to wrap
the middleware framework itself--which then caused problems with other things
assuming that functions were created from some prototype.

I also had a terrible time getting the layout engine working. Error reporting
was difficult to trace, and the middleware indirection made debugging difficult.
I tried for days to get the templating plugin working and could not work it out.
And [I couldn't convince](https://github.com/jonschlinkert/gray-matter/issues/34)
the owners of the metadata plugin that parsing YAML data files while
assuming they always had frontmatter was a bad idea
(there's actually a syntax conflict there!).

So nice idea but try as I might, it was too much work and I let that iteration
of my blog die.

## Attempt 3: Hugo

A good year later, I'd heard and seen some positive things about Hugo.

- Hugo is fast.
- Hugo is simply just cross platform. One .exe (for Windows) and go.
- There is no tool chain. Just one .exe and go.
- Hugo is super flexible. 
  - Nested static pages just work
  - Collections (like blog posts) have a higher level abstraction that can be
    moulded into anything you like.

There is one large drawback. Since it is written in Go, it is compiled, and
essentially impossible to customize. However, with very few exceptions Hugo has
pretty much everything you need built in, like:

- code highlighting
- templating (with a weird, but capable templating language)
- support for a range of content formats
- support for themes

If you don't need any custom CSS or JavaScript assets then you're good to go.
However I did want that flexibility so I stole a webpack build off of the default
theme.

Speaking of themes, after trying to customise the default theme, I realised that
it was just easier to not use a theme. So that is what I've done.

It's still early days but it looks like I'm happy with Hugo. I might write a little
more about some of the customisations I made later.


