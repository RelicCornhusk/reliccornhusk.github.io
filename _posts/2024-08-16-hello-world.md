---
layout: post
title: "Hello, World: blogging with Jekyll + GitHub Pages"
subtitle: "A DevOps engineer's approach to blogging"
tags: GitHub
giscus_comments: true
---

`print("Hello, World")`

Hey there, you found my blog! This is a space I'll be using to share a bit of what I do at work and my side projects, as well as what technologies and tools I've been learning about.

This first post will be a bit meta. I'll share my experience so far playing with the website generator Jekyll and publishing on GitHub Pages. This approach to blogging makes a lot of sense if you're already comfortable with writing in Markdown, you have experience with Git and GitHub and enjoy using CLI tools (or would like to gain experience at that). It could even be a nice project for someone just starting out on their developer journey, allowing you to practice your skills with GitHub, Docker (and even a bit of Ruby) while also creating a platform to write about what you're learning.

## [GitHub Pages](pages.github.com)

GitHub Pages is a way to host a static website for free directly out of a GitHub repository. This means you can simply commit to that repository, and it takes care of deploying the website for you using some predefined GitHub Actions workflows. It even provisions you a TLS certificate through Let's Encrypt. Free users are allowed one website per account, and it supports custom domains at no extra cost. It checked all the boxes for me in terms of hosting platform, especially since I enjoy working with GitHub Actions. I've since learned there are other great options to consider, such as [GitLab Pages](https://docs.gitlab.com/ee/user/project/pages/) and [CloudFlare Pages](https://pages.cloudflare.com/).

## [Jekyll](jekyllrb.com)

After that, I needed a tool for generating the blog itself and this is where Jekyll comes in. Jekyll is a blog-aware static website generator built on Ruby. It allows you to blog directly through Markdown files and to manage, build and serve your website through simple CLI commands. This means you don't have to worry about managing a database as you would with WordPress: it looks for certain YAML and Markdown files to generate the HTML for your website. Simply running `jekyll serve` will host your website locally, allowing you to preview on a browser your changes almost in real time.

There are other great options for static website generators. [Hugo](gohugo.io) is an even more performant alternative to Jekyll which is built on Go. I haven't had the time to try it, but I recommend [this post](https://cloudcannon.com/blog/jekyll-vs-hugo-choosing-the-right-tool-for-the-job/) for a comparison between the two.
Here is what made me go with Jekyll:

- **Simple setup**. The Jekyll CLI is intuitive to use and has commands to facilitate the management of your files. So apart from changing my ruby version with rbenv and managing the dependencies with Bundler (nice skills to have picked up), there wasn't much else I needed to do to get up and running. You could even skip setting up a local environment altogether depending on the Jekyll theme chosen. That is because a lot of themes come with custom GitHub Actions workflows that remove the need for installing Jekyll and any of its plugins locally. With [the theme I'm currently using](https://github.com/alshedivat/al-folio), you can simply push a commit and wait a few minutes while GitHub Actions builds and publishes your website with Jekyll. That works well for a hands-off approach, but I would still recommend setting up Jekyll locally to be able to play with the Jekyll command line and to preview your website before deploying. Many themes come with a Docker image for making that part seamless.

- **Widely adopted by an active community**. This was the most important factor for me. If you browse the options for Jekyll themes, you will find a plethora to choose from built by the community, some free and some paid. There are a lot of amazing paid themes options [here](https://jekyllthemes.io/) for fair prices, which come with instructions on how to configure to your liking. If you're looking for a free theme, simply searching GitHub for "Jekyll theme" and sorting by number of stars will give you great options to choose from.

## Conclusion

After a week or so playing with Jekyll and GitHub Pages to create this website, I would highly recommend it for the technology oriented of you out there who want to get into blogging or build a portfolio website for free. As a bonus, it taught me a little bit about interacting with Ruby projects and it was my first experience playing with Visual Studio Code's Dev Containers—interesting on paper but quite laggy, think I'll just stick to Podman! Most importantly, it saves you the trouble of dealing with HTML or managing a WordPress database. Of course, despite it being free (_if you don't count my fancy domain_), it's clear that there's a trade-off in terms of the time and skills required with this approach versus simply paying for a service that builds a webpage for you through a user-friendly UI. The best option for you will be determined by how comfortable and willing you are to get 'hands-on' with the solution you choose.
