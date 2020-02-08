---
layout: post
title:  "Example"
date:   2020-02-08 18:03:20 +0000
categories: dummy
img:
  screenshot.png
---
You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.


Example of ruby snippet:
{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Example of putting an image using base url:

![]({{site.baseurl}}/img/screenshot.png)

Example of code snippet:
```
code?
```



{{site.url}}


Link to [My Site][my-site]

[my-site]: https://kevinnlei.github.io/site/
