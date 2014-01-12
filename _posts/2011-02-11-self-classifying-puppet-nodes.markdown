---
layout: post
title:  "Self-classifying puppet nodes"
date:   2011-02-11 11:48:13
categories: puppet operations
---

Puppet has a  very cool node classification system which pretty much lets you do what you want (by writing your own one) if the default classifier doesn't work for you. So, there are already a couple of good posts around this, and its worth reading some of the following posts: <a href="http://www.semicomplete.com/blog/geekery/puppet-nodeless-configuration.html">Jordan Sissel</a> , <a href="http://glarizza.posterous.com/our-puppet-external-node-infrastructure"> Gary Larizza </a> as well as the official docs on <a href="http://docs.puppetlabs.com/guides/external_nodes.html">external node classifiers</a>.

So, from the above posts, I'm going to take a few of the ideas, mix them up, and go through the steps to reproduce on your own system. The goal of the end configuration is to have a node come online, identify itself using it's Role, Platform, and Environment; and then issue it the relevant classes. What's important here, is that nodes must be classified, before they reach puppet, into Roles and Platforms (as well as Environment, but this is already handled by puppet). Dividing nodes by their Platform/Role gives us the simplicity needed when you're managing a large number of machines across different clusters. Its easier to group machines than it is to individually assign classes to each node. Of course, not all your puppetized nodes need to belong to a group as they might just be one machine performing a specific action. In cases like this, we must be able to add exceptions easily.

For the purpose of this post, let's assume we have 2 clusters in Europe and USA, and each cluster has several Application and Web Servers. I'm also assuming you're following the recommended puppet-mcollective-facter setup, because it works well.

From a high level overview, we want to write a facter plugin for mcollective which will read a facts file on the host. This facts file will contain the Role and Platform information that can be used from Puppet. We then need an mcollective agent so we can update this file if we need to at a later stage. Finally we look at how to create node classification system that can use these facts to hand out the right manifest.

<strong>Identification</strong>

Facter is the game, and we need a new fact. So facter gives puppet access to information about a host at run-time like what country a host is in or what distribution of linux it's running. We're going to put 2 new facts, and for the sake of best practices, we'll make it extensible. I don't like polluting the existing facter namespace with odd names, and so i'm going to prefix all facts with a name (use your company name or whatever you want).

The following is a facter plugin that will parse the file <em>/etc/company.facts</em> and append them to the existing facts.

{% highlight ruby %}

require 'facter'

if File.exist?("/etc/company.facts")
    File.readlines("/etc/company.facts").each do |line|
        if line =~ /^(.+)=(.+)$/
            var = "company_"+$1.strip;
            val = $2.strip

            Facter.add(var) do
                setcode { val }
            end
        end
    end
end

{% endhighlight %}

Given the following facts file <em>/etc/company.facts</em>:

{% highlight ruby %}
Role = Web
Platform = USA
{% endhighlight %}

We will get the following from facter

{% highlight ruby %}
...
company_role = Web
company_platform = USA
...
{% endhighlight %}

These variables are now available straight away in your puppet manifests.

<strong>Updating the Facts</strong>
Before i continue on using these facts in puppet, its important to have a way to update the facts. Equally important is that you implement the facts into your server deploy process. So, we have a script that installs mcollective and puppet when we commission a new server, and one of the first things that is done is to create this file and automatically populate the Role and Platform based on the commissioning paramaters.

Apart from server deploy-time, we can write a small mcollective RPC agent which will get/set/delete values from our facts file. The file has a simple key-value structure and so the following should do the job

{% highlight ruby %}
module MCollective
    module Agent
        class Companyfact<RPC::Agent
            metadata    :name       => "Company Fact Agent",
                    :description    => "Key/values in a text file",
                    :author     => "Puppet Master Guy",
                    :license    => "GPL",
                    :version    => "Version 1",
                    :url        => "www.company.com",
                    :timeout    => 10

            companyfile = "/etc/company.facts"

            def parse_facts(fname)
                begin
                    if File.exist?(fname)
                        kv_map = {}
                        File.readlines(fname).each do |line|
                            if line =~ /^(.+)=(.+)$/
                                @key = $1.strip;
                                @val = $2.strip
                                kv_map.update({@key=>@val})
                            end
                        end
                        return kv_map
                    else
                        f = File.open(fname,'w')
                        f.close
                        return {}
                    end
                rescue
                    logger.warn("Could not access company facts file. There was an error in companyfacts.rb:parse_facts")
                    return {}
                end
            end

            def write_facts(fname, facts)

                if not File.exists?(File.dirname(fname))
                   Dir.mkdir(File.dirname(fname))
                end

                begin
                    f = File.open(fname,"w+")
                    facts.each do |k,v|
                        f.puts("#{k} = #{v}")
                    end
                    f.close
                    return true
                rescue
                    return false
                end
            end

            action "get" do
                validate :key, String

                kv_map = parse_facts(companyfile)
                if kv_map[request[:key]] != nil
                    reply[:value] = kv_map[request[:key]]
                end
            end

            action "put" do
                validate :key, String
                validate :value, String

                kv_map = parse_facts(companyfile)
                kv_map.update({request[:key] => request[:value]})

                if write_facts(companyfile,kv_map)
                    reply[:msg] = "Settings Updated!"
                else
                    reply.fail!  "Could not write file!"
                end

            end
            action "delete" do
                validate :key, String

                kv_map = parse_facts(companyfile)
                kv_map.delete(request[:key])

                if write_facts(companyfile,kv_map)
                    reply[:msg] = "Setting deleted!"
                else
                    reply.fail!  "Could not write file!"
                end

            end
        end
    end
end

{% endhighlight %}

We also need the ddl:

{% highlight ruby %}
metadata        :name           => "Company Fact Agent",
        :description    => "Key/values in a text file",
        :author         => "Puppet Master Guy",
        :license        => "GPL",
        :version        => "Version 1",
        :url            => "www.company.com",
        :timeout        => 10

action "get",   :description => "fetches a value from a file" do
    display :failed

    input :key,
        :prompt     => "Key",
        :description    => "Key you want from the file",
        :type       => :string,
        :validation => '^[a-zA-Z0-9_]+$',
        :optional   => false,
        :maxlength  => 90

    output :value,
        :description    => "Value",
        :display_as => "Value"
end

action "put", :description = "Value to add to file" do
    display :failed

    input :key,
        :prompt     => "Key",
        :description    => "Key you want to set in the file",
        :type       => :string,
        :validation => '^[a-zA-Z0-9_]+$',
        :optional   => false,
        :maxlength  => 90

    input :value,
                :prompt         => "Value",
                :description    => "Value you want to set in the file",
                :type           => :string,
                :validation     => '^[a-zA-Z0-9_]+$',
                :optional       => false,
                :maxlength      => 90

    output :msg,
        :description    => "Status",
        :display_as => "Status"
end

action "delete", :description = "Delete a key/value pair if it exists" do
        display :failed

        input :key,
                :prompt         => "Key",
                :description    => "Key you want to change in the file",
                :type           => :string,
                :validation     => '^[a-zA-Z0-9_]+$',
                :optional       => false,
                :maxlength      => 90

        output :msg,
                :description    => "Status",
                :display_as     => "Status"
end

{% endhighlight %}

For a quick refresh on using your mc-rpc agent, we can set a key using the following:
<code>mc-rpc -v --agent companyfact --action put --argument key=role --argument value=Web</code>

And we can get a key using the following
<code>mc-rpc -v --agent companyfact --action get --argument key=role</code>

And we can delete a key using the following
<code>mc-rpc -v --agent companyfact --action delete --argument key=role</code>

<strong>Self-Classifying Nodes</strong>
This is where we want to be. A node comes in and says to puppet, I'm a Web machine on platform USA.

The default basic setup is to use a node definition for each node, or plug some sort of external classifier on. I'm going to build on from Jordan Sissel's blog that I mentioned at the start. Essentially, every node goes through the 'default' node definition, which then goes to the 'truth enforcer'. This truth enforcer will look at the facts of the node and hand off the relevant classes accordingly. <em>Note</em> that if you want to add exceptions, just create a node definition for the exception node. simple.

So the enforcer node is a very basic definition:

{% highlight ruby %}
node default {
  include truth::enforcer
}
{% endhighlight %}

From here, we create a truth enforcer class like so (using our example). <em>Naturally this is just an example of how it might be used</em>:

{% highlight ruby %}
class truth::enforcer {

        $groupname = "$company_platform:$company_role"
        case $groupname {
                "USA:Web" : {
                        include roles::web
                }
        }

        case $company_role {
                "Application" : {
                        include roles::application
                }
        }
}
{% endhighlight %}

That's pretty much it as far as getting a self-classifying puppet node goes. One more thing that's worth mentioning is that this also ties in well with Extlookup to manage your parameters. You can use something like the following configuration which I find works well:

{% highlight ruby %}
$extlookup_precedence = ["fqdn_%{fqdn}", "role_%{company_role}-%{company_platform}", "platform_%{company_platform}", "common"]
{% endhighlight %}

[jekyll]:    http://jekyllrb.com
