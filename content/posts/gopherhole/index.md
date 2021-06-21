---
title: "Emerging from my Gopher Hole"
date: 2021-06-21T10:30:00-04:00
description: Emerging from my Gopher hole
hero: hero.jpeg
menu:
  sidebar:
    name: Emerging from my Gopher Hole
    weight: 20
---

## Inspiration

I was inspired recently after listening to [Go Time FM episode #167](https://changelog.com/gotime/167) and [Johnny Boursiquot](https://www.jboursiquot.com/)'s [callout](https://changelog.com/gotime/167#t=50:41):

> "If you are out there, and I'm speaking to you listener, or watcher, if you are out there and thinking/meaning to muster up the energy or overcome the imposter syndrome to write a blog post that's beginner, but you're thinking "Oh man, everyone's already written a beginner blog post on this thing, my voice doesn't really matter I'm not going to really add anything to that" - you need to get over that. There's always room for new ideas. There's always room for new thinking, new approaches. Right, even if you think you're rehashing something else somebody has said, I haven't read your take on that thing. I've read dozens more, but I don't mind reading another one, so get over that fear, right - put something out there. Whatever form or shape that you'd like, just contribute your part of the story. We want your part of the story, so contribute that." - Johnny Boursiquot

## Reflection

When I started with Go in 2016, it seemed like the expected path to learning was to read the standard library and paste knowledge together from various blog posts, books, etc. In-person trainings were limited and expensive. From experience though, I've found that trying to pick up Go knowledge through just reading and coding is extremely inefficient. It's the combination of "digging in the docs" and putting yourself "out there" that seems to be the magic combination.

If you're like me though, putting yourself "out there" in the technology arena can be terrifying. I have a degree in Hotel & Restaurant Management. I spent years in hotels and restaurants until I fell into IT in 2000. Ever since, I've been writing code in everything from Cobol to Java to now Go as well as worked my way into management, but through all that time and experience - without a Computer Science degree - I still feel out of place. In my early days of Go learning, I always felt outclassed somehow, as if I wasn't technical enough. There was, to me, a sense that by reading the standard library and with enough experience, I would eventually start writing "idiomatic" code. I had no way of knowing exactly when I had reached this point of enlightenment, but I told myself I'd eventually get there. The reality though, is that Johnny is right, I need to get over that - I've wasted far too much time trying to get there. It doesn't help that I've been writing Go code now for more than 4 years, but with the exception of a few short, quick conversations, I've never actually talked to another Go developer about Go! Honestly. I've had a few random chats about Go at a Meetup where I felt completely awkward and had paralyzing imposter syndrome, but in general, I've worked alone at home, in my spare time, trying to learn Go by reading everything I could get my hands on. I've read many of the Go books, most of the blog posts out there and have tried contributing where I could in open source projects to get some exposure and learnings.

I have been working my way to this point all along though and through all of it, the Go community though has been really fantastic - every time I've reached out for help in a forum, I've found it. Every time I tried to contribute to an open source library, the help has been welcomed. Each of those interactions were actually exponentially more helpful than just reading posts, so thank you to all those out there trying to do their part in the community. I now realize that everything I publish does not have to be perfect or even guaranteed to be the most mature idea possible as every engineer innately understands that time and experience make for better code. That's the beauty of putting yourself "out there" actually - you're giving examples of your humanity in the code you write. [Mat Ryer](https://twitter.com/matryer), for instance, I have been following and learning from him for years, and his coding has evolved in the open - take his "How I write Go HTTP services after *n* years" posts - he's annually updating and refining his code publicly in writing - it's great!

## Template/Boilerplate Project

I was attracted to Go originally because of it's welcoming, inclusive community and Go's design ethos. See [this Rob Pike post](https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html) for what I mean by the latter. In particular, this quote:

> "Go was designed to help write big programs, written and maintained by big teams."

I spent 20+ years working with very large teams in my former job, designing and working with really large codebases and databases. I love designing and building processes and code templates for teams that make their lives easier, so Go is a natural fit for me.

I had big dreams when I started my template/boilerplate repo [https://github.com/gilcrest/go-api-basic](https://github.com/gilcrest/go-api-basic) - I wrote this [silly little post](https://dangillis.dev/posts/archive/go-api-template/) on October 20, 2017 for `v0.0.2`:

> Hello Gopher community! This represents my first code release, albeit a minor one. I’m hopeful that this will be helpful. I believe in templates and am trying to put together a working API template from all the best practices I’ve found in the Go community.

My idea was to create a boilerplate/template repo that was a fully functioning web server implementation with all the scaffolding needed for large teams and projects (security, logging, tests, db integration, etc.) using as few frameworks as possible. At the time, there weren't a ton of these - there are a bunch now and they all bring different points of view! I named it `go-api-basic` as I originally had ideas for a basic and an advanced version.

*Guess what?!* It's almost 4 years later and I'm ***STILL*** working on the basic version! Yikes. It's been evolving over the years and has taken various shapes - at one point when modules were first introduced, I tried to make *EVERYTHING* a module. I separated everything out into multiple repos and tried to make it super modular and highly composable. It was a productivity nightmare. At another point, I tried to do a Go version of "clean code" and had a very layered, complex structure that also ended up feeling unproductive. I've tried going with the "flat approach", which is the opposite of that, but that didn't quite work for me either. I'm now somewhere in the middle of all that, but what I have I like. I'm positive I'll make more changes and hopefully, after I get feedback from other humans, it will get even better, but I would really love feedback. Give me negative feedback! Give me positive feedback! I really don't care, I would love to get other perspectives on it.

I'm going to write a series of blog posts that describe the project and then fold those posts into the project `README`. I just launched my new site [dangillis.dev](https://dangillis.dev/) (using [HUGO](https://gohugo.io/) of course!) and my first new post (not counting this) is [https://dangillis.dev/posts/errors/](https://dangillis.dev/posts/errors/) - about how I do error handling in the project. I'm planning a series of fast-followers after.

Stay tuned and thanks!

Dan
