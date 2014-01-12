---
layout: post
title:  "Puppet: pulling a list of hosts from foreman"
date:   2010-10-25 11:48:13
categories: puppet foreman
---

I'm on a mission to completely automate the configuration of munin by taking the excellent work of <a href="http://github.com/duritong/puppet-munin">duritong on puppet-munin</a>, and embellishing it a little in order to create an Overview section which can aggregate a cross-section of hosts. One of the main enablers for this will be a little-known wrapper function that comes with puppet-foreman.

The function provided is a 'Query Interface' to foreman, and queries foreman via an API, which returns a list of hosts. This wrapper function actually makes a call via a ruby script provided with the foreman installation. As <a href="http://theforeman.org/wiki/foreman/Query_Interface">the docs</a> say, you'll have to modify line 8 of this script to point to your foreman installation.

If you would like to test that the function is pointing correctly and working, create a hostgroup, attach to a host, and use the following curl:

{% highlight bash %}
curl -v http://foreman?hostgroup=somehostgroup&format=yml&state=all
{% endhighlight %}

If all is good you'll see something like

{% highlight bash %}
---
  - hostname.domain
{% endhighlight %}

The next thing we want to do is use the puppet-foreman provided (example) function, <em>foreman</em>, to make a call for us and return a list of hosts. To do this, we just need to import the foreman module into our namespace and make the call. The call comes standard as taking a number of optional parameters as follows:
<p style="padding-left:30px;">hostgroup=groupname : list hosts that are in the group</p>
<p style="padding-left:30px;">fact=factname : list hosts with a specific fact.</p>
<p style="padding-left:30px;">class=classname : list hosts with a class attached</p>
<p style="padding-left:30px;">verbose=yes : output the hosts facts as well</p>
A simple class that implemented this might look like:

{% highlight ruby %}

class test {
    $hostlist = split(gsub(foreman("hostgroup=cluster1"),"[ -]*",""),'\n')
    file {
        ... some template that uses hostlist
    }
}

{% endhighlight %}

where some template could be as simple as listing the fqdn of the hosts that match your query.<em> Later on we will have a look at how to use the facts supplied from clients in a central manifest.</em>

{% highlight ruby %}

<% hostlist.each do |host| %>
    <%= host %>
<% end >%

{% endhighlight %}

The code for the manifest is a bit obscure and I came across it in some old mail archive's. Essentially, the output from the wrapper function is a string and so it needs to get passed through the split and gsub functions provided in the common module. The split function returns an array which we can then use to pass into our template. This is something we'll look into just now. 

Something that caught me out initially, is that if you are trying to list hosts by classname, the classes have to be directly attached, as opposed to implicitly via a hostgroup. e.g. If you have a node belonging to a hostgroup which has class x. A search for hosts with class x will not return the node.

The example wrapper that has been given in the docs is a little basic and has some elements missing which are available in the actual Query Interface. For instance, when specifying a fact, you can only specify whether a host contains has that fact, and not whether it has a fact with a specific value. 

<strong>A better puppet parser function</strong>
Looking at some missing elements from the query interface, there a couple improvements we would like to get from the wrapper function:
<ol>
    <li> Be able to return a list of hosts with fact=value.</li>
    <li> Return an array / hash instead of a string and use those facts given in a verbose response</li>
    <li> Option to include all hosts irrespective of their state</li>
</ol>

Here is my code for the updated wrapper function based on the original with some modifications.
<code>/etc/puppet/modules/foreman/lib/puppet/parser/functions/foreman.rb </code>

{% highlight ruby %}

require 'net/http'

# Query Foreman
module Puppet::Parser::Functions
 newfunction(:foreman, :type => :rvalue) do |args|
    #URL to query
    host = "foreman"
  url = "/hosts/query?"
  query = []
  args.each do |arg|
    name, value, fact_val = arg.split("=")
    case name
    when "fact"
      query << "#{name}=#{value}-seperator-#{fact_val}"
    when "class", "hostgroup"
      query << "#{name}=#{value}"
    when "verbose"
      query << "verbose=yes" if value == "yes"
    when "state"
      query << "state=all" if value == "all"  
    else
      raise Puppet::ParseError, "Foreman: Invalid parameter #{name}"     
    end   
end   

begin
     response = Net::HTTP.get host,url+query.join("&")+"&format=yml"
rescue Exception => e
    raise Puppet::ParseError, "Failed to contact Foreman #{e}"
end

  begin
    hostlist = YAML::load response
  rescue Exception => e
    raise Puppet::ParseError, "Failed to parse response from Foreman #{e}"
  end
  return hostlist
 end
end

{% endhighlight %}


With this modified wrapper function in place (after running a pluginsync) we can simplify our class to:

{% highlight ruby %}

class test {

    $hostlist_cluster1 = foreman("hostgroup=cluster1", "state=all")
    $hostlist_os = foreman("fact=operatingsystem=Ubuntu", "state=all")

    file {
        ... some template that uses hostlist
    }
}

{% endhighlight %}

where some template could be

{% highlight ruby %}

<% hostlist.each do |host| %>
    <%= host %>
<% end >%

{% endhighlight %}


Now, if you're using puppet 2.6, you can also use the verbose option to return hosts and their facts which just adds another level of awesomeness. The verbose output adds in the meta information to the YAML, and if you do a curl as follows:
{% highlight bash %}
curl -v http://foreman?hostgroup=somehostgroup&format=yml&state=all&verbose=yes

#You can also try parse through ruby to get a cleaner output:
curl -v 'http://foreman?hostgroup=somehostgroup&format=yml&state=all&verbose=yes' > /tmp/f.list
cat /tmp/f.list | ruby -e 'require "yaml"; y=STDIN.read; y=y.gsub("!ruby/sym ",""); print YAML.dump(YAML.load(y))'

{% endhighlight %}

You should see a yours hosts coming back with this kind of structure

{% highlight bash %}
---
   - host.domain
        - facts
             - somefact=x
             - somefact=x
        - classes
             - someclass
{% endhighlight %}

So, to use this information from the updated foreman function, we use verbose as:
{% highlight ruby %}

    $hostlist = foreman("fact=operatingsystem=Ubuntu", "state=all", "verbose=yes")

{% endhighlight %}

An in our template, we can reference the facts:
{% highlight ruby %}

<% hostlist.each do |host| %>
    <%= host['facts']['fqdn'] %> runs <%= host['facts']['operatingsystem'] %>
<% end >%

{% endhighlight %}

This final piece of the puzzle, bringing facts and meta-data into the picture, will enable fairly complex setups for distributed systems with a centralized configuration. There are other options and I'd encourage anyone reading this to get familiar with <a href='http://www.devco.net/archives/2009/08/31/complex_data_and_puppet.php'>ExtLookup</a> from RI Pienaar, as well as the exported resources which are available as part of the core.
