+++
title = "Hello, world!"
date = "2015-05-28 08:29:03"
description = "Hello, world!  The beginning of Mirango.io"
draft = false
+++

Hello, world!  I've decided it's time to restart my blog.

I'm a computer science student and programmer living in Boston with interests in functional programming and concurrent computing.

I've had several blogs in the past - mostly self hosted wordpress blogs.  Wordpress offers a lot of functionality that I don't need for a simple blog, so I was excited to learn about static site generators.

Since my blog will be serving identical pages, there's really no need for a backend.  Rather than storing posts and pages in a database, a static site generator, as the name suggests, creates static HTML from a mix of templates, markdown, and css.  For mirango.io I decided to use [Jekyll][jekyll].

Feel free to take a look at the source for Mirango.io.  It's available on [GitHub][mirango] along with the Nginx [configuration file][nginx].  If you're interested in using Jekyll yourself, I've provided a short installation guide below.


<h4>Short guide to installing Jekyll on your local computer</h4>

```bash
# Install rvm and ruby (stable)
$ curl -sSL https://get.rvm.io | bash -s stable --ruby

# Install the jekyll gem
$ gem install jekyll

# Create a new jekyll site (jekyll will create the basic file structure and configuration files)
$ jekyll new siteName

# Move into the newly created file
$ cd siteName

# Builds the site (takes all the files and outputs the static site in the _site folder)
$ jekyll build

# Creates a development web server at localhost:4000
$ jekyll serve
```

If you're interested in creating a jekyll site, GitHub will host the site for free.  Check out [https://pages.github.com][pages].

[mirango]: https://www.github.com/sdwalsh/mirango
[nginx]: https://www.github.com/sdwalsh/mirango_nginx
[pages]: https://pages.github.com
[jekyll]: http://jekyllrb.com
