---
layout: post
title:  "Chef execution ordering"
date:   2011-09-30 11:48:13
categories: chef
---

Chef is really powerful because it is written as a layer on top of an interpreted language (ruby). So when you come up against a limitation in the ddl, you are able to dig into the layer beneath the ddl and manipulate objects directly. I want to focus on some of the things i've found useful when writing recipes that have let me force execution order, or at least understand it's boundaries.

<strong>Execution order of ruby code</strong>

The first thing to understand about writing a recipe is how it is interpreted. When a resource is declared (package, file, ...), the resource is tacked onto a list and, as the cookbooks are evaluated, the list grows until. When all cookbooks have been evaluated, the resources in the list are executed in order. So how will code that is outside of a resource block run? What we find is that the resource's base class does the job of appending itself to this list, and so if you put ruby code outside of a resource block, it will run before the entire list is executed. Just to make sure this is clear, lets take a look at the following recipe:

{% highlight ruby %}

package 'apache2' do
  action :install
end

Chef::Log.info("Apache2 installed")

{% endhighlight %}

The initial thought might be that the package will be installed and then you will see a log message "Apache2 installed". What really happens is that the package resource is evaluated, and appended onto the resource list. The log statement is run and outputs "Apache2 installed", and finally, the package is installed when the resource list is evaluated.

<strong>Ruby blocks</strong>

So how do we make sure our ruby is run after a specific recipe? Ruby blocks are maybe a little under utilized, they simply let us execute ruby code while taking advantage of the base resource class methods.

{% highlight ruby %}

ruby_block 'ruby-logstuff' do
  block do
    Chef::Log.info("write some stuff out")
  end
end

{% endhighlight %}

There is another way, but I haven't done much looking into it. Simply attach ruby into the resource object. It works, but i'm not sure if the code will run before the resource actions, or after (probably before).

{% highlight ruby %}

package 'apache2' do
  action :install
  Chef::Log.info("Apache2 installed")
end

{% endhighlight %}

<strong>Forcing a resource to run immediately</strong>

We may want a resource to run before anything else in any cookbook. Usually you would seperate this kind of thing out into a separate recipe so that you can ensure it runs first. However, sometimes we may want to keep methods grouped together because its logical. Every resource returns a reference to itself, and this lets us call it! The following is a silly impractical example...

{% highlight ruby %}

# this will run 2nd
package 'haproxy' do
  action :install
end

# this will run first
package 'apache2' do
  action :nothing
end.run_action(:create)

{% endhighlight %}

