---
layout: post
title:  "Bringing the dl380 (hypervisor) under puppets control"
date:   2014-12-26 20:47:00
categories: puppet
tags:
  - Puppet Enterprise
  - puppet
  - forge
---
Getting the hypervisor under puppets control
===========================================

Installing the PE 3.7.1 agent on the dl380 (hypervisor)
=======================================================

Since the OS of my puppet master and the dl380 are the same, installing the puppet agent is quit simple :

  {% highlight bash %}
    curl -k https://pepuppet:8140/packages/current/install.bash | sudo bash.
  {% endhighlight %}

If you are running this command as root, one can leave the sudo from the command. Above command will install the pe-agent package and all its dependencies, and start the pe-puppet process :

  {% highlight bash %}
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100  9805  100  9805    0     0  28264      0 --:--:-- --:--:-- --:--:-- 28338
    Loaded plugins: fastestmirror, langpacks
    base                                                     | 3.6 kB     00:00
    extras                                                   | 3.4 kB     00:00
    puppetlabs-pepackages                                    | 2.5 kB     00:00
    updates                                                  | 3.4 kB     00:00
    puppetlabs-pepackages/primary_db                         |  33 kB   00:00
    Loading mirror speeds from cached hostfile
     * base: mirror.kinamo.be
     * extras: mirror.kinamo.be
     * updates: mirror.kinamo.be
    Resolving Dependencies
    Running transaction check
    ---> Package pe-agent.noarch 0:3.7.1-1.pe.el7 will be installed

    <......>

    Notice: /Service[pe-puppet]/ensure: ensure changed 'stopped' to 'running'
    service { 'pe-puppet':
      ensure => 'running',
      enable => 'true',
    }
  {% endhighlight %}

Now, when we trigger a puppet run, we encounter the following message :

  {% highlight bash %}
    [root@dl380 ~]# /opt/puppet/bin/puppet agent -t --noop
    Exiting; no certificate found and waitforcert is disabled
    [root@dl380 ~]#
  {% endhighlight %}

meaning we need to sign the dl380's the certificate on the puppetmaster.
Let's first have a look at the certificates that needs to be signed :

  {% highlight bash %}
    [root@pepuppet ~]# puppet cert list
       "dl380.koewacht.net" (SHA256) FA:6C:58:69:A2:E9:D7:EF:74:69:31:BF:2C:77:90:25:00:D0:6F:A4:AE:A1:A8:62:F1:7A:80:6D:EA:71:40:BE
    [root@pepuppet ~]#
  {% endhighlight %}

Signing can be done with (on the puppetmaster)

  {% highlight bash %}
    [root@pepuppet ~]# puppet cert sign dl380.koewacht.net
    Notice: Signed certificate request for dl380.koewacht.net
    Notice: Removing file Puppet::SSL::CertificateRequest dl380.koewacht.net at '/etc/puppetlabs/puppet/ssl/ca/requests/dl380.koewacht.net.pem'
    [root@pepuppet ~]#
  {% endhighlight %}

Last step to do is to trigger a run on dl380, to install the Puppet Enterprise Agent configuration stuff.

  {% highlight bash %}
    root@dl380 ~]# /opt/puppet/bin/puppet agent -t
    Info: Caching certificate for dl380.koewacht.net
    Info: Caching certificate_revocation_list for ca
    Info: Caching certificate for dl380.koewacht.net

    <......>

    Info: /Stage[main]/Puppet_enterprise::Mcollective::Server/File[/etc/puppetlabs/mcollective/server.cfg]: Scheduling refresh of Service[pe-mcollective]
    Notice: /Stage[main]/Puppet_enterprise::Mcollective::Service/Service[pe-mcollective]/ensure: ensure changed 'stopped' to 'running'
    Info: /Stage[main]/Puppet_enterprise::Mcollective::Service/Service[pe-mcollective]: Unscheduling refresh on Service[pe-mcollective]
    Notice: Finished catalog run in 1.86 seconds
  {% endhighlight %}

and now our dl380 is a real puppet node. When we go to the [console][console], we see two nodes, our puppetmaster and the dl380.  Another simple way to check if everything is correctly configured is using mcollective.  Just follow the next steps on the puppetmaster :

  {% highlight bash %}
    [root@pepuppet ~]# su - peadmin
    peadmin@pepuppet:~$ mco ping
    pepuppet.koewacht.net                    time=113.79 ms
    dl380.koewacht.net                       time=120.94 ms

    ---- ping statistics ----
    2 replies max: 120.94 min: 113.79 avg: 117.37
    peadmin@pepuppet:~$
  {% endhighlight %}

The Puppet Forge
================
But now what ? I get the feeling I will need many more Virtual Machines, and I'm not the guy to repeat this boring process over and over again.  If we look at how we did install the puppet master,we need the following :

* A storage-pool
* A volume in that storage pool for the new VM
* A kickstart file for installation
* A virtual guest image definition

I see some candidates of new puppet type, the storage pools and volumes, some templates for the kickstart file and the domain.xml, and a good candidate to be used for provider, and then I'm looking towards libvirt.

This is already a good list of requirements to start with.  So lets have a look if something useful already exists.

  {% highlight bash %}
    peadmin@pepuppet:~$ puppet module search libvirt
    Notice: Searching https://forgeapi.puppetlabs.com ...
    NAME                  DESCRIPTION                                                         AUTHOR        KEYWORDS
    thias-libvirt         Libvirt virtualization API and capabilities                         @thias        centos kvm rhel libvirt debian
    example42-libvirt     Puppet module for libvirt                                           @example42    libvirt example42
    desalvo-libvirt       Puppet module for libvirt management                                @desalvo      kvm libvirt
    gaudenz-libvirt       Puppet module to install libvirt and create virtual domain conf...  @gaudenz      virtualization drbd kvm libvirt
    carlasouza-virt       Virt Module for Puppet                                              @carlasouza   kvm xen openvz libvirt
    domcleal-katellovirt  Installs a libvirt hypervisor for testing provisioning              @domcleal     libvirt foreman katello
    jhoblitt-ksm          Manages Linux Kernel Samepage Merging (KSM)                         @jhoblitt     ksm kvm libvirt qemu virt rhel
    tuxomatic-vkick       Module to define KVM virtual machines and their OS configuratio...  @tuxomatic    fedora rhel centos libvirt kvm
  {% endhighlight %}

Hmm, but which one does meet the most my requirements ?  Looking at the description, my first selection is :

* thias-libvirt
* example42-libvirt
* desalvo-libvirt
* gaudenz-libvirt
* tuxomatic-vkick

Next shifting of the candidates, looking at the modules on the puppet forge.

Lets  arrange them based on some statistics on the forge :

* thias-libvirt     : Version 0.3.2 • May 5, 2014   • Downloads 2986 •  Current version 947
* gaudenz-libvirt   : Version 1.1.2 • Dec 15, 2014  • Downloads 244  •  Current version  57
* desalvo-libvirt   : Version 0.1.2 • Nov 7, 2014   • Downloads 515  •  Current version 113
* tuxomatic-vkick   : Version 0.3.0 • Jul 28, 2014  • Downloads 354  •  Current version 268
* example42-libvirt : Version 2.0.9 • Oct 25, 2013  • Downloads 734  •  Current version 575


Seems we need to have a closer look at the modules itself, by looking at there individual Puppet Forge Page.

Following modules did not qualify :

* gaudenz-libvirt : only supports debian
* desalvo-libvirt : to basic
* example42-libvirt : A great module, lake all examples42 ones, but to complex for my setup I believe.

So I still have two good candidates :

* tuxomatic-vkick : seems to offer enough functionality to delve deeper into the module
* thias-libvirt   : that looks like the one.  Has support for networking, storage pools etc

And peeking into the modules, I believe I will go with :

* thias-libvirt

testing a puppet forge module
=============================

In the control repo, (the one that contains the Puppetfile, manifests etc ...), we will create a new branch,
add the thias/libvirt modules to the Puppetfile in this branch.  Also time to create our profiles and roles modules, so we are able to introduce this concept in our setup and add both new modules also to the Puppetfile.

  {% highlight bash %}
    [witjoh@fc20 pepuppet]$ git branch thias
    [witjoh@fc20 pepuppet]$ git branch
    * production
      thias
    [witjoh@fc20 pepuppet]$ git checkout thias
    Switched to branch 'thias'
    [witjoh@fc20 pepuppet]$ git branch
      production
    * thias
    [witjoh@fc20 pepuppet]$ vi Puppetfile 
    [witjoh@fc20 pepuppet]$ git status
    On branch thias
    Changes not staged for commit:
      (use "git add <file>..." to update what will be committed)
      (use "git checkout -- <file>..." to discard changes in working directory)

      modified:   Puppetfile

    no changes added to commit (use "git add" and/or "git commit -a")
    [witjoh@fc20 pepuppet]$ git add Puppetfile
    [witjoh@fc20 pepuppet]$ git commit -m "Added libvirt mod, and my roles and profiles modules"
    [thias 18c15b5] Added libvirt mod, and my roles and profiles modules
     1 file changed, 5 insertions(+), 5 deletions(-)
    [witjoh@fc20 pepuppet]$ git push origin thias
    Counting objects: 5, done.
    Delta compression using up to 2 threads.
    Compressing objects: 100% (3/3), done.
    Writing objects: 100% (3/3), 370 bytes | 0 bytes/s, done.
    Total 3 (delta 2), reused 0 (delta 0)
    To https://github.com/witjoh/pepuppet.git
     * [new branch]      thias -> thias
    [witjoh@fc20 pepuppet]$
  {% endhighlight %}

I created an empty repository on github, and cloned them on my laptop.  In those empty repos, we will create an empty puppet module.

  {% highlight bash %}
    [witjoh@fc20 koewacht-newpuppet]$ git clone https://github.com/witjoh/pepuppet-profiles.git
    Cloning into 'pepuppet-profiles'...
    warning: You appear to have cloned an empty repository.
    Checking connectivity... done.
    [witjoh@fc20 koewacht-newpuppet]$ git clone https://github.com/witjoh/pepuppet-roles.git
    Cloning into 'pepuppet-roles'...
    warning: You appear to have cloned an empty repository.
    Checking connectivity... done.
    [witjoh@fc20 koewacht-newpuppet]$ ls

  {% endhighlight %}

For the roles/profiles modules, I just created two empty repositories on GitHub, cloned them on my laptop, and used the
'puppet module generate' command to populate both repository with the puppet module tree structure.  I did also some cleanup and
adjusted the README.md files.

After that, is pushed those changes to github :

  {% highlight bash %}
[witjoh@fc20 pepuppet-profiles]$ git push origin master
  {% endhighlight %}

And next thing is to run r10k on the puppet master, which should create a new 'thias' environment :

  {% highlight bash %}
    [root@pepuppet ~]# r10k deploy environment -pv
    [R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/production
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/production/modules/motd
    [R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/thias
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/libvirt
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/motd
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/roles
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/profiles
  {% endhighlight %}

Oh my god, I hope this does not comply to my setup :
  {% highlight plain %}
    Hi,

    I really hope I am mistaken, but it seems you can't use r10k with it's full potential anymore in Puppet Enterprise 3.7.

    In the past (PE3.3) I could create any branch, feature_something, in git repository. r10k would create a branch/environment
    and I could apply it from the command line on some development host, i.e.

    puppet agent -t --environment=feature_something

    Now, according to puppetlabs support, "console node classifier being the definitive and authoritative environment-setter", so I can't do that anymore.
    This seems like a major setback and a deal breaker for me. Does anybody else use environments this way?
    Why even have --environment switch altogether if I can't select anything besides what is set in the console.

    What are other choices except ditching PE altogether?

    Thanks,
    Vadym
  {% endhighlight %}

Lets see if we run in the same issue, as following [Did PE 3.7 kills r10k?][post] mailing list describes

  {% highlight plain %}
    [root@dl380 ~]# puppet agent -t --noop --environment=thias
    Warning: Local environment: "thias" doesn't match server specified node environment "production", switching agent to "production".
    Info: Retrieving pluginfacts
    Info: Retrieving plugin
    Info: Loading facts
    Info: Caching catalog for dl380.koewacht.net
    Info: Applying configuration version '4f106fbbda0a37b186e403fd442f304f26a4b9c1'
    Notice: Finished catalog run in 0.64 seconds
    [root@dl380 ~]# grep environment /etc/puppetlabs/puppet/puppet.conf
        environment = thias
    [root@dl380 ~]#
  {% endhighlight %}

Even is we put the environment in the puppet.conf, the '[agent]' section, or overrule it on the command line, puppet refuses to switch to the
environment we want, and stick with the 'production' environment. Reading the docs and the mailing lists, this is the new behaviour starting form puppet enterprise 3.7.  Assigning classes to nodes has undergone a dramatic change,  From now on, the console is one and only place where a node is coupled to a specific environment, which is by default, production.

And the solutions can be find in the reply of Justin :

  {% highlight plain %}
    The new classifier does allow agents to specify their own environments, but it involves an extra couple of steps. What you'll want to do is create a new group for the nodes that you're concerned with (which could be all of them) and set the environment of the new group to "agent-specified" in the pull-down menu. If you never want to override that setting with another classifier group, then you should also set "Override all other environments" in the group's metadata.
  {% endhighlight %}

Time to start reading some docs  :

* [The release notes of PE3.7][release]
* [The new Node Manager App][classification]

The PE console - making r10K environments work
=============================================

The only thing we want to achieve right now is to make it possible to switch environments without adjusting some
configuration, be it in the console, puppet.conf or site.pp or some other place where it could be possible. Thing is we only have one single
hypervisor, so we don't have the luxury to play with the puppet code and configuration on some testing machine.

Later on, we will look at some testing frameworks and methods, but for now, my priority is to get everything in puppet asap so we can deploy.
a lot of virtual machines using puppet.

And yes, this is risky, but I hope I have documented everything in such detail we can easily start from scratch.

Let's create a new node group, hypervisor, because that is the role of the dl380 we want to manage by puppet.
After logging in into the PE console, we select the menu 'classification'.

![Classification Screen][classification_img]

Just under the Node Group name, enter the name 'hypervisor', and then click on the 'add group' button.
When we click on our new group, we get the following screen :

![Classification: Group detail][classification_detail_img]

At the top, we have a link 'Edit node group metadata', that's our next stop :

![Classification: Group metadata][classification_meta_img]

Here we have added a description, and changed the environment to 'agent-specified'.  I also ticked 'override all other environments', since r10k manages  all modules I need in a specific environment.  If other modules needs some changes to make the whole thing work, then this will not conflict with the other modules, and I can test those changes before promoting them to the other environments.

At the bottom of the screen, we need to 'commit 1 change'.

Now, we have to add nodes to this group.  We could use the 'pin; feature, where we have the ability to assign one specific node directly to this group.  Looks like that is similar to the old way. But lets use the rule based node classification. Lets look at a fact that we can use to select the type of hardware we use for hypervisor. (since I only have one piece of hardware, this is kind of stupid, but we need to get used to the new way of classification).

![Classification: Group Rules][classification_rules_img]

Curious which node is selected :

![Classification: Nodes selected][classification_node_img]


and the final test :

  {% highlight bash %}
    [root@dl380 ~]# puppet agent -t --noop --environment=thias
    Info: Retrieving pluginfacts
    Info: Retrieving plugin
    Info: Loading facts
    Info: Caching catalog for dl380.koewacht.net
    Info: Applying configuration version 'da0ee0ff523209960433174e8c161d2c83e3b86e'
    Notice: Finished catalog run in 0.80 seconds
    [root@dl380 ~]#
  {% endhighlight %}

One thing that we still need : adding the classes to the group :

In a first step to get my hypervisor stuff, especially the libvirt thing into puppet, I just create a new profile,
and start adding stuff one by one.   Where this will end,  I don't really know, and it is not that important right know.
By refactoring over and over again, we will hopefully come to a workable solution.



[post]: https://groups.google.com/d/msgid/puppet-users/CA%2BQfOKiQG%2BZd-kt_U1tcvOrK2yE9-JXR9P0jZYM%2Bdhp_6Q8NRA%40mail.gmail.com?utm_medium=email&utm_source=footer
[console]: https://your.puppetmaster.somewhre/
[release]: https://docs.puppetlabs.com/pe/latest/install_upgrading_notes.html#upgrading-puppet-enterprise:-notes-and-warnings
[classification]: https://docs.puppetlabs.com/pe/latest/console_classes_groups.htmlhttps://docs.puppetlabs.com/pe/latest/console_classes_groups.html

[classification_img]: {{ site.url }}/images/classification_img.png "Classification Screen"
[classification_detail_img]:{{ site.url }}/images/classification_detail_img.png "Classification Group Detail"
[classification_meta_img]: {{ site.url }}/images/classification_meta_img.png "Classification Group Meta Data"
[classification_rules_img]: {{ site.url }}/images/classification_rules_img.png "Classification Group Rules"
[classification_node_img]: {{ site.url }}/images/classification_node_img.png "Classification Node Selected"
