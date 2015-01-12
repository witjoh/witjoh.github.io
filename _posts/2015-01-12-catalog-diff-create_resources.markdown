---
layout: post
title:  "The create_resources() function ans catalog-diff issues"
date:   2015-01-12 13:23:00
categories: puppet
tags:
  - catalog-diff
  - hiera
  - create_resources
---
The *create_resources()* function and *catalog-diff* issues
=======================================================

While introducing hiera at one of our customers, we were looking at a way to move some of the 'data'
from the manifests into *hiera*.  And the first thing that came to mind was [zack's catalog-diff][catalog-diff].

Whatever we changed in function of the *hiera* integration should compile the same catalog, otherwise we did something wrong. And for regular resources, we saw indeed when we did the conversion the right way, or when we did make a mistake somewhere.

But a function that is often used when working with *hiera*, is the [create_resources()][create_resources]. And what we saw while moving some data to a *hiera* hash, and used the *create_resources()* function, was that *catalog-diff* does not recognise them as a *resource* in the catalog.  After *catalog-diff*, they does not exist in the *new*, only in the *old* catalog.  It is not *hiera* related, but really has something to do with the *create_resources()* function.
Lets have a closer look, using a very simple example.  We do have two environments setup for this exercise, old and new.

In a node_test.pp we have the following resource, which is the *old* version :

  {% highlight puppet %}

    node 'test_node' {
      notify { 'my_test_notify':
        message  => 'this is a test notify',
        name     => 'test_notify',
        withpath => true,
      }
    }
  {% endhighlight %}

In the new version, we will be using a the *create_resources()* to achieve the same result :

  {% highlight puppet %}
    node 'test_node' {
      $my_notify =  { 'my_test_notify' => { 'message' => 'this is a test notify',
                                            'name' => 'test_notify',
                                            'withpath' => 'true', } }
      create_resources(notify, $my_notify)
    }
  {% endhighlight %}

Both simple puppet code generates the same catalog  :

  {% highlight json linenos %}
     .......
     {
       "parameters": {
         "withpath": true,
         "message": "this is a test notify",
         "name": "test_notify"
       },
       "exported": false,
       "line": 12,
       "file": "/etc/puppetlabs/puppet/environments/old/manifests/nodes/test_node.pp",
       "tags": [
         "notify",
         "my_test_notify",
         "node",
         "test_node",
         "class"
       ],
       "title": "my_test_notify",
       "type": "Notify"
     }
     .......
  {% endhighlight %}

and the one with the *create_resources()* generates the following catalog.

  {% highlight json linenos %}
    {
      "parameters": {
        "withpath": "true",
        "message": "this is a test notify",
        "name": "test_notify"
      },
      "exported": false,
      "tags": [
        "notify",
        "my_test_notify",
        "node",
        "test_node",
        "class"
      ],
      "title": "my_test_notify",
      "type": "Notify"
    }
  {% endhighlight %}

But this is what a *catalog-diff* gives  :

  {% highlight shell-session %}
    Catalog diff output :
    --------------------------------------------------------------------------------
    test_node                                                                 100.0%
    --------------------------------------------------------------------------------
    Total resources in old: 1
    Total resources in new: 0
    Only in old:
        notify[my_test_notify]
    Catalag percentage added:       0.00
    Catalog percentage removed:     100.00
    Catalog percentage changed:     0.00
    Added and removed resources:    0 / -1
    Node percentage:        100.0
    Node differences:       1
    --------------------------------------------------------------------------------
    1 out of 1 nodes changed.                                                 100.0%
    --------------------------------------------------------------------------------
    Nodes with the most changes by percent changed:
    1. test_node                                                             100.00%
    Nodes with the most changes by differences:
    1. test_node                                                                  1false
  {% endhighlight %}

As you see in both catalogs experts, the notify resources are the same.  The only difference lies in the extra parameters, which do not have any impact on the resource itself.  For every resource created with the *create_resources()* function, the *"files":* and *"lines":* meta attribute is missing. And both those attributes are used to display the file and line in the error message when in the event of a catalog compilation failure.

The following function from the *<modulepath>/catalog-diff/lib/puppet/catalog-diff/preprocessor.rb* is responsible to see if the element from the catalog is a resource or not :

Time to have a look at some code from 

  {% highlight ruby lineno %}
    *converts Puppet 0.25 and 2.6.x catalogs to our intermediate format
    def convert25(resource, collector)
      if resource.is_a?(Puppet::Resource::Catalog)
        resource.edges.each do |b|
          convert25(b, collector)
        end
      elsif resource.is_a?(Puppet::Relationship) and resource.target.is_a?(Puppet::Resource) and resource.target.title and resource.target.file
        target = resource.target
        manifestfile = target.file.gsub("/etc/puppet/manifests/", "")

        resource = {:type => target.type,
          :title => target.title,
          :parameters => {}}

        target.each do |param, value|
          resource[:parameters][param] = value
        end

        if resource[:parameters].include?(:content) and resource[:parameters][:content] != false
          resource[:parameters][:content] = { :checksum => Digest::MD5.hexdigest(resource[:parameters][:content]), :content => resource[:parameters][:content] }
        end

        resource[:resource_id] = "#{target.type.downcase}[#{target.title}]"
        collector << resource
      end
    end
  {% endhighlight %}

Look at the elsif condition :

* resource.target.file

And that is not set in the *new* catalog.  And if we have a look at the *create_reource.rb* code :
(located in Puppet Enterprise at this location : /opt/puppet/lib/ruby/site_ruby/1.9.1/puppet/parser/functions/create_resources.rb )

  {% highlight ruby lineno %}
    Puppet::Parser::Functions::newfunction(:create_resources, :arity => -3, :doc => <<-'ENDHEREDOC') do |args|
        Converts a hash into a set of resources and adds them to the catalog.

        This function takes two mandatory arguments: a resource type, and a hash describing
        a set of resources. The hash should be in the form `{title => {parameters} }`:

            # A hash of user resources:
            $myusers = {
              'nick' => { uid    => '1330',
                          gid    => allstaff,
                          groups => ['developers', 'operations', 'release'], },
              'dan'  => { uid    => '1308',
                          gid    => allstaff,
                          groups => ['developers', 'prosvc', 'release'], },
            }

            create_resources(user, $myusers)

        A third, optional parameter may be given, also as a hash:

            $defaults = {
              'ensure'   => present,
              'provider' => 'ldap',
            }

            create_resources(user, $myusers, $defaults)

        The values given on the third argument are added to the parameters of each resource
        present in the set given on the second argument. If a parameter is present on both
        the second and third arguments, the one on the second argument takes precedence.

        This function can be used to create defined resources and classes, as well
        as native resources.

        Virtual and Exported resources may be created by prefixing the type name
        with @ or @@ respectively.  For example, the $myusers hash may be exported
        in the following manner:

            create_resources("@@user", $myusers)

        The $myusers may be declared as virtual resources using:

            create_resources("@user", $myusers)

      ENDHEREDOC
      raise ArgumentError, ("create_resources(): wrong number of arguments (#{args.length}; must be 2 or 3)") if args.length > 3
      raise ArgumentError, ('create_resources(): second argument must be a hash') unless args[1].is_a?(Hash)
      if args.length == 3
        raise ArgumentError, ('create_resources(): third argument, if provided, must be a hash') unless args[2].is_a?(Hash)
      end

      type, instances, defaults = args
      defaults ||= {}

      resource = Puppet::Parser::AST::Resource.new(:type => type.sub(/^@{1,2}/, '').downcase, :instances =>
        instances.collect do |title, params|
          Puppet::Parser::AST::ResourceInstance.new(
            :title => Puppet::Parser::AST::Leaf.new(:value => title),
            :parameters => defaults.merge(params).collect do |name, value|
              Puppet::Parser::AST::ResourceParam.new(
                :param => name,
                :value => Puppet::Parser::AST::Leaf.new(:value => value))
            end)
        end)

      if type.start_with? '@@'
        resource.exported = true
      elsif type.start_with? '@'
        resource.virtual = true
      end

      begin
        resource.safeevaluate(self)
      rescue Puppet::ParseError => internal_error
        raise internal_error.original
      end
    end
  {% endhighlight %}

As far as I can see, no resource.file and/or resource.line is set by the *create_resources*.

But question now, should we patch the catalog-diff *preprocessor.rb* or puppet's native *create_resources.rb*

That's what I try to find out right now.

TBC

[catalog-diff]: https://forge.puppetlabs.com/zack/catalog_diff
[create_resources]: https://docs.puppetlabs.com/references/latest/function.html#createresources
