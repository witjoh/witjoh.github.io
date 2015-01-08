---
layout: post
title:  "Exploring The Tiny Puppet Module of Alessandro Franceschi"
date:   2015-01-06 13:23:00
categories: puppet
tags:
  - Tiny Puppet
  - puppet
  - forge
---
Exploring The Tiny Puppet Module of Alessandro Franceschi
=========================================================

Installing the module with r10k
-------------------------------

The module itself can be found on [example42 puppet-tp on github][tp].

I added the module to my 'thias' environment using r10k. One could also create a new environment, but since I still have to do a lot of stuff (well, I have to puppetize almost everything), maybe I can integrate the -Tiny Puppet- module right away. First thing to do is to add the module to our Puppetfile on our workstation (read laptop).

  {% highlight shell-session %}
    [witjoh@fc20 pepuppet]$ git branch
      production
      * thias
    [witjoh@fc20 pepuppet]$ vi Puppetfile
  {% endhighlight %}

We added the following entry to the *Puppetfile*

  {% highlight json %}
    ....

    # getting some hands on with tiny::puppet
    mod 'puppet-tp',
      :git => 'https://github.com/example42/puppet-tp.git'

    ....
  {% endhighlight %}

And we need to push our change to the *thias* branch on the remote git repository :

  {% highlight shell-session %}
    [witjoh@fc20 pepuppet]$ git status
    On branch thias
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git checkout -- <file>..." to discard changes in working directory)

      modified:   Puppetfile

    no changes added to commit (use "git add" and/or "git commit -a")
    [witjoh@fc20 pepuppet]$ git add Puppetfile
    [witjoh@fc20 pepuppet]$ git commit -m "added  tiny puppet module from Alessandro for review"
    [thias c1a5333] added  tiny puppet module from Alessandro for review
     1 file changed, 4 insertions(+)
    [witjoh@fc20 pepuppet]$ git push origin thias
    Username for 'https://github.com': witjoh
    Password for 'https://witjoh@github.com': ********
    Counting objects: 7, done.
    Delta compression using up to 2 threads.
    Compressing objects: 100% (3/3), done.
    Writing objects: 100% (3/3), 418 bytes | 0 bytes/s, done.
    Total 3 (delta 2), reused 0 (delta 0)
    To https://github.com/witjoh/pepuppet.git
       da0ee0f..c1a5333  thias -> thias
  {% endhighlight %}

And on our puppet master, we need to run the *r10k* command

  {% highlight shell-session %}
    [witjoh@fc20 pepuppet]$ ssh pepuppet
    johan@pepuppet's password: *********
    [johan@pepuppet ~]$ sudo -i
    [root@pepuppet ~]# r10k deploy environment -pv
    [R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/production
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/production/modules/motd
    [R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/thias
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/libvirt
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/motd
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/roles
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/profile
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/tp
  {% endhighlight %}

And it never hurts to verify the result :

  {% highlight shell-session %}
    [root@pepuppet ~]# /opt/puppet/bin/puppet module list --environment=thias
    Warning: Missing dependency 'puppetlabs-stdlib':
      'witjoh-profiles' (v0.0.1) requires 'puppetlabs-stdlib' (>= 1.0.0)
      'witjoh-roles' (v0.0.1) requires 'puppetlabs-stdlib' (>= 1.0.0)
    Warning: Missing dependency 'puppetlabs-stdlib':
      'thias-libvirt' (v0.3.2) requires 'puppetlabs-stdlib' (>= 2.0.0)
      'example42-tp' (v0.1.0) requires 'puppetlabs-stdlib' (>= 3.2.0 < 5.0.0)
    /etc/puppetlabs/puppet/environments/thias/modules
    ├── example42-tp (v0.1.0)
    ├── motd (???)
    ├── thias-libvirt (v0.3.2)
    ├── witjoh-profiles (v0.0.1)
    └── witjoh-roles (v0.0.1)
    /etc/puppetlabs/puppet/modules (no modules installed)
    /opt/puppet/share/puppet/modules
    ├── puppetlabs-pe_accounts (v2.0.2-6-gd2f698c)
    ├── puppetlabs-pe_concat (v1.1.2-4-g2b7bba2)
    ├── puppetlabs-pe_console_prune (v0.1.1-4-g293f45b)
    ├── puppetlabs-pe_inifile (v1.1.4-16-gcb39966)
    ├── puppetlabs-pe_java_ks (v1.2.4-35-g44fbb26)
    ├── puppetlabs-pe_postgresql (v3.4.4-15-g32e56ed)
    ├── puppetlabs-pe_razor (v0.2.1-9-g8d78ec2)
    ├── puppetlabs-pe_repo (v0.7.7-51-g5ba0427)
    ├── puppetlabs-pe_staging (v0.3.3-2-g3ed56f8)
    └── puppetlabs-puppet_enterprise (v3.7.1-90-g4a9e885)
  {% endhighlight %}

But that is not exactly what I expected to see. Luckily this does not mean I have a partial PE 3.7.1 installation.
This is confirmed by [Walid on the pe-user mailing list][thread] :

  {% highlight text %}
    Yes, PE 3.7.1 went with a better model for including the minimum out of the box modules so that one has freedom to
    pick and choose, also when you choose a puppet enterprise supported module you will see the naming is prefixed with
    pe_ so that there are no name space conflicts with other in-house or forge modules
  {% endhighlight %}

And is is also documented in the stdlib module documentation :

  > Note: As of version 3.7, Puppet Enterprise no longer includes the stdlib module.
  > If you're running Puppet Enterprise, you should install the most recent release of stdlib
  > for compatibility with Puppet modules.

We already knew it, but this proves that r10k is not handling module dependencies automatically.  And that is how I
want it.

So we need to add stdlib to our puppetmaster. I do have the choice now, do i install the puppetlabs/stdlib module with r10k, or do we install it manually wit the *puppet module install ....* command ?

Going for the first option, we have the possibility to include or exclude the puppetlabs/stdlib module for specific environments, couple versions to other environments or whatever combination one can think on.  But in this case, because we are talking about the one module everybody is using, I  will go for the second option for the initial install of this module. When upgrades are needed, then I will use the first option, because I will have (hopefully) coded my infrastructure using puppet DSL, and then I really want to test the upgrade path.

But what is the best place to install modules manually for all environments ?  Lets have a look at the modulepath for my current environments (keep in mind the default environment in puppet is 'production') :

  {% highlight shell-session %}
    [root@pepuppet modules]# /opt/puppet/bin/puppet master --configprint modulepath
    /etc/puppetlabs/puppet/environments/production/modules:/etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules
    [root@pepuppet modules]# /opt/puppet/bin/puppet master --configprint modulepath --environment=thias
    /etc/puppetlabs/puppet/environments/thias/modules:/etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules
    [root@pepuppet modules]#
  {% endhighlight %}

The first directory is managed by r10k, so that's not an option.  The last directory is used for the puppet enterprise specific modules.  And I do not want to touch those modules. So the '/etc/puppetlabs/puppet/modules' looks a good option. Check the *basemodulepath* in the *[main]* section of your */etc/puppetlabs.puppet/puppet/conf* configuration file. Time for action :

  {% highlight shell-session %}
    [root@pepuppet puppet]# puppet module install puppetlabs-stdlib -i /etc/puppetlabs/puppet/modules/
    Notice: Preparing to install into /etc/puppetlabs/puppet/modules ...
    Notice: Downloading from https://forgeapi.puppetlabs.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppetlabs/puppet/modules
    └── puppetlabs-stdlib (v4.5.0)
    [root@pepuppet puppet]#
  {% endhighlight %}

I'm using the *-i <install directory>* option, to override the default installation directory, being the first one if the *modulepath* configuration settings.  Now we have the stdlib module available in all current and future environments :

  {% highlight shell-session %}
    [root@pepuppet puppet]# puppet module list
    /etc/puppetlabs/puppet/environments/production/modules
    └── motd (???)
    /etc/puppetlabs/puppet/modules
    └── puppetlabs-stdlib (v4.5.0)
    /opt/puppet/share/puppet/modules
    ├── puppetlabs-pe_accounts (v2.0.2-6-gd2f698c)
    ├── puppetlabs-pe_concat (v1.1.2-4-g2b7bba2)
    ├── puppetlabs-pe_console_prune (v0.1.1-4-g293f45b)
    ├── puppetlabs-pe_inifile (v1.1.4-16-gcb39966)
    ├── puppetlabs-pe_java_ks (v1.2.4-35-g44fbb26)
    ├── puppetlabs-pe_postgresql (v3.4.4-15-g32e56ed)
    ├── puppetlabs-pe_razor (v0.2.1-9-g8d78ec2)
    ├── puppetlabs-pe_repo (v0.7.7-51-g5ba0427)
    ├── puppetlabs-pe_staging (v0.3.3-2-g3ed56f8)
    └── puppetlabs-puppet_enterprise (v3.7.1-90-g4a9e885)

    [root@pepuppet puppet]# puppet module list  --environment=thias
    /etc/puppetlabs/puppet/environments/thias/modules
    ├── example42-tp (v0.1.0)
    ├── motd (???)
    ├── thias-libvirt (v0.3.2)
    ├── witjoh-profiles (v0.0.1)
    └── witjoh-roles (v0.0.1)
    /etc/puppetlabs/puppet/modules
    └── puppetlabs-stdlib (v4.5.0)
    /opt/puppet/share/puppet/modules
    ├── puppetlabs-pe_accounts (v2.0.2-6-gd2f698c)
    ├── puppetlabs-pe_concat (v1.1.2-4-g2b7bba2)
    ├── puppetlabs-pe_console_prune (v0.1.1-4-g293f45b)
    ├── puppetlabs-pe_inifile (v1.1.4-16-gcb39966)
    ├── puppetlabs-pe_java_ks (v1.2.4-35-g44fbb26)
    ├── puppetlabs-pe_postgresql (v3.4.4-15-g32e56ed)
    ├── puppetlabs-pe_razor (v0.2.1-9-g8d78ec2)
    ├── puppetlabs-pe_repo (v0.7.7-51-g5ba0427)
    ├── puppetlabs-pe_staging (v0.3.3-2-g3ed56f8)
    └── puppetlabs-puppet_enterprise (v3.7.1-90-g4a9e885)
    [root@pepuppet puppet]#
  {% endhighlight %}


[tp]: https://github.com/example42/puppet-tp
[thread]: https://groups.google.com/a/puppetlabs.com/forum/#!topic/pe-users/_KAgwwwzJQI
