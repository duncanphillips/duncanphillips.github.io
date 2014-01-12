---
layout: post
title:  "Puppet: pulling a list of hosts from mcollective"
date:   2011-01-11 11:48:13
categories: puppet mcollective
---

In one of my <a href="http://abigant.wordpress.com/2010/10/25/pulling-a-list-of-hosts-from-foreman-for-puppet/">previous posts</a>, I wrote about using foreman (a node classifier and dashboard for puppet) to retrieve a list of hosts (and meta information) so that you can use it in a puppet manifest. We're looking to do something like iterate over a set of nodes in a template (I use it for generating the Munin server config). Stored resources provide a way to centralize information from nodes, but it isn't very intuitive, and gets a little tricky to plan and maintain. 

While foreman is good for a lot of use-cases, not everyone uses it. So, I want to provide an alternative for those that don't use foreman. The alternative uses MCollective to populate a list of hosts based on information given to us by the MC Registration plugin. Before I dive in, I'd like to quickly cover off a blog from the MCollective architect, R.I.Pienaar, on this very topic. His <a href="http://www.devco.net/archives/2010/09/18/puppet_search_engine_with_mcollective.php">blog post (PuppetSearch)</a> brings together 3 things. MongoDB, MC Registration, and Puppet in a very powerful way. It is a really good solution, is more robust, and we've since moved on to something similar. The following blog is more just to help understand how flexible puppet can be, and how well it integrates with MCollective. The advantage of using PuppetSearch is that you can load a specific node, and you can query using MongoDB syntax.

So, we're looking to achieve something quite simple really. We want a subset of hosts matching a basic query, with their meta information, in a variable in a puppet manifest.

<strong>MCollective and Registration</strong>
MCollective is great, and very flexible. One of the core plugins for MCollective is called registration. Essentially, every node/host sends a registration message at a pre-determined interval set in the configuration. The registration message is sent as a broadcast, and so any client can pick up the registration messages of any other client. We only want one registration handler, but its nice to know that there can be more than one handler (1 per node). 

The registration message can be anything, and in our case, we want to send the client identifier, it's facts, and maybe you want to add some more information along the way. This message is picked up by a handler, which then processes the message. 

Currently, we've taken the lead of R.I.Pienaar (from his blog above) and shoved the messages straight into a MongoDB instance so that we can query it and use it for different parts of our operations infrastructure. Since that's done, I'm going to cover off the plain old text file version. It's extremely similar in architecture, but the code does completely different things.

<em>Installing the registration plugin</em> isn't too difficult. We need to do it in 2 parts which is pretty standard and has been well documented.

Part 1 is getting the clients to register with all the information they have, including facts, and anything else you like. For this, one of the <a href="https://github.com/puppetlabs/mcollective-plugins/blob/master/registration/meta.rb">documented registrations</a> will do just fine. This file will sit in your MCollective <em>plugins/mcollective/registration</em> directory. You then need to adjust your config file for the server to say what registration plugin you're using and how often.

<em>server.cfg</em>
{% highlight text %}
registerinterval = 300
registration = Meta
{% endhighlight %}

After this, you should have registration messages flying around. You need to handle them. The <a href="http://projects.puppetlabs.com/projects/mcollective-plugins/wiki/AgentRegistrationMonitor">RegistrationAgent</a> provides a simple handler which will write the messages to a text file per client. check_mcollective is optional, and we won't be using it. It provides a link to nagios if you wish to explore. So, for the client we want to handle the registration messages (probably your mcollective/puppet server), we want to put <a href="https://github.com/puppetlabs/mcollective-plugins/blob/master/agent/registration-monitor/registration.rb">registration.rb</a> in <em>MCollective plugins/mcollective/agent</em>.

Restart MCollective on all nodes that have been affected, and you should start seeing the registrations text files populating in <em>/var/tmp/mcollective/</em>. You can change this directory by specifying it in the client.cfg on the node where the handler is. (<code>plugin.registration.directory = '/etc/mcollective/registered/'</code>)

Ok, so that's MCollective registration done. If you want to add more information, just make some changes to meta.rb.

<strong>Hostlist (Puppet function)</strong>

Ok, here we come onto the real topic which is to use what we've implemented above to retrieve a list of hosts with their facts. The function itself is very easy because all the information we have is already in YAML and we can just load it, and spit it out!

So the function below is a puppet parser function which can be used in manifests.  Please excuse my Ruby...

{% highlight ruby %}

# mc_hostlist.rb
# Duncan Phillips

# Retrieve a list of hosts and their meta information by querying the data stored by the registration agent.
# info on the registration agent can be found at http://marionette-collective.org/reference/plugins/registration.html

# Usage: mc_hostlist([class],[fact])
# If neither is specified, all hosts are returned.
# Class and Fact are filters and can both be specified
# Fact can be specific or non-specific. i.e. machine with fact, or machine with fact=z

# e.g. mc_hostlist(class=hosting, fact=operatingsystem=Ubuntu)
#[hostname => {facts : {fact1 : value1}, classes : {class1 : value 1}}]

require 'yaml'

module Puppet::Parser::Functions
    newfunction(:mc_hostlist, :type => :rvalue) do |args|
        #populate our array/map
        hosts = Dir.entries("/var/tmp/mcollective")
        for h in hosts do
            begin
                if (h == '.') or (h == '..')
                    hosts[hosts.index(h)] = nil; 
                else
                    hfile = open("/var/tmp/mcollective/"+h)
                    raw = hfile.read.gsub("!ruby/sym ","")
                    hosts[hosts.index(h)]=YAML.load(raw).merge({"fqdn"=>h})
                end
            rescue Exception => e
                raise Puppet::ParseError, "There was an exception: " + e + "\n"
            end
        end

        args.each do |arg|

            name, value, factvalue = arg.split("=")

            case name
            when "fact"
                hosts=hosts.compact
                for h in hosts do
                    if hosts[hosts.index(h)]["facts"][value]
                        if (factvalue) and (hosts[hosts.index(h)]["facts"][value] != factvalue)
                            hosts[hosts.index(h)]=nil
                        end
                    else
                        hosts[hosts.index(h)]=nil
                    end
                end
            when "class"
                hosts=hosts.compact
                for h in hosts do
                    if hosts[hosts.index(h)]["classes"].index(value) == nil
                        hosts[hosts.index(h)]=nil
                    end
                end
             
            else
                raise Puppet::ParseError, "mc_hostlist: Invalid parameter #{name}"
            end #case
        end #args
    
        return hosts.compact

    end #func
end
{% endhighlight %}

This is a puppet parser, and so it needs to be installed into a module as such. You can find out more about this <a href="http://docs.puppetlabs.com/guides/plugins_in_modules.html">here</a>. I recommend just putting it into the common module, in which case it will go into MCollective <em>modules/common/lib/puppet/parser/functions/</em>. After this you'll need to resync the plugins (usually a puppet run will suffice if pluginsync is turned on in the configs).

<strong>Into the manifest</strong>

So, how do we use this? I'm going to give some insight into how one can use this to generate a Munin conf file... I won't go into other bits, but will look at what's relevant here.

Below is an example of how one might use a list of all hosts which have the class '<em>Web</em>' to create an aggregate graph. We can aggregate anything, for now we'll create a graph of the load for every node in the list.

{% highlight puppet %}

    $hl_web  = mc_hostlist("class=Web")

    file {  "/etc/munin/munin.conf": content => template("munin/munin.conf"), }

{% endhighlight %}

We can then use this in the template file as below (Once again, excuse my Ruby):

{% highlight ruby %}

<% hl_web.each do |h| -%>

# Register the nodes

[GroupName;<%= h['fqdn'] %>]
        address <%= h['fqdn'] %>
        use_node_name yes
<% end %>

# Create a new group Totals which holds aggregate graphs

[Totals; GroupName]

        # Generate our aggregate graph

        web_load.graph_title Load Average
        web_load.graph_category GroupName
        web_load.graph_scale no
        web_load.graph_vlabel Load
        web_load.graph_order \<% hl_web.each_with_index do |h,i| -%><% if (h != '') %>
                <%= h['fqdn'][/[a-zA-Z0-9]*/] %>=GroupName;<%= h['fqdn'] %>:load.load <% if i != hl_web.size-1 -%>\<% end -%><% end -%><% end %>
        <% hl_web.each_with_index do |h,i| %>
        <% if (h != '') -%>web_load.<%= h['fqdn'][/[a-zA-Z0-9]*/] %>.draw LINE1<% end %>
        <% end %>

{% endhighlight %}

<strong>End result:</strong> A nice aggregate graph that will dynamically add hosts as they register.
