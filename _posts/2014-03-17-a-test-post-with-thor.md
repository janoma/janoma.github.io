---
layout: post
title: "A test post with Thor"
tags: [github-pages, gems, thor]
category: programming
---

I found [this little
tool](http://jonasforsberg.se/2012/12/28/create-jekyll-posts-from-the-command-line)
that helps create a post for [github-pages](http://pages.github.com/) with [Jekyll](http://jekyllrb.com/) from the command
line. It creates the file with the appropriate URL-friendly name and it opens
your choice editor so you can start writing your content immediately.

## Usage

Make sure you have the `thor` and `stringex` gems, then create a `jekyll.thor`
file in the root directory of your github-pages project with the following
content:

{% highlight ruby linenos %}
require "stringex"
class Jekyll < Thor
    desc "new", "create a new post"
    method_option :editor, :default => "vim"
    def new(*title)
    title = title.join(" ")
    date = Time.now.strftime('%Y-%m-%d')
    filename = "_posts/#{date}-#{title.to_url}.md"

    if File.exist?(filename)
        abort("#{filename} already exists!")
    end

    puts "Creating new post: #{filename}"
    open(filename, 'w') do |post|
        post.puts "---"
        post.puts "layout: post"
        post.puts "title: \"#{title.gsub(/&/,'&amp;')}\""
        post.puts "tags: []"
        post.puts "category: general"
        post.puts "---"
    end

    system(options[:editor], filename)
    end
end
{% endhighlight %}

I modified it so it uses `vim` as the default editor, the tags are a
one-liner list and there is a default category.
