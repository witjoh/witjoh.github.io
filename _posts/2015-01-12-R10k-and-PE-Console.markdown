---
layout: post
title:  "Dynamic Environments, R10K, and Puppet Enterprise Node Classification"
date:   2015-01-12 23:05:00
categories: puppet
tags:
  - Puppet Enterprise
  - R10K
  - Dynamic Environments

---
Introduction
------------
Playing a bit with Puppet Enterprise, I noticed the classification has changed a lot.
In the old days we assigned classes, or a group of classes to a node directly.  Now
we assign classes to **nodegroups**, which are then populated with nodes by defining **rules**.

Another change in the latest Puppet Enterprise, is how the environment is handled by puppet.
If an environment is set in the PE console, then that environment is authoritative.  You cannot overrule it,
neither on the command line, or by setting the **environment** in your **/etc/puppetlabs/puppet/puppet.conf**.

And when one wants to use R10K to create environments on the fly, they will have big troubles switching environments due to the environment
behaviour.

Lets delve into the node classification in PE and look how to make it work.

Classification in Puppet Enterprise 3.7
---------------------------------------
What I want to do is to test a module from **the forge**, so I in my **master repo** I created a new branch, which I called
**thias**, after the module I want to review : [**thias/libvirt**][thias-libvirt].

I added this module to my **Puppetfile**, pushed it to the master repo, and ran **R10K** on my puppet master.

I already created the **hypervisor** group, which is described in [this post][prev_post].

Looking at my current setup, I have the following environments setup :

  {% highlight shell-session %}
    .
    |-- production
    |   |-- environment.conf
    |   |-- hieradata
    |   |   |-- common.yaml
    |   |   |-- dev.yaml
    |   |   |-- prod.yaml
    |   |   `-- test.yaml
    |   |-- hiera.yaml
    |   |-- manifests
    |   |   `-- site.pp
    |   |-- modules
    |   |   `-- motd
    |   |-- Puppetfile
    |   `-- README.md
    `-- thias
        |-- environment.conf
        |-- hieradata
        |   |-- common.yaml
        |   |-- dev.yaml
        |   |-- prod.yaml
        |   `-- test.yaml
        |-- hiera.yaml
        |-- manifests
        |   `-- site.pp
        |-- modules
        |   |-- libvirt
        |   |-- motd
        |   |-- profiles
        |   |-- roles
        |   `-- tp
        |-- Puppetfile
        `-- README.md
  {% endhighlight %}

And in the PE console, I added the hypervisor group, as shown in next image :

![Overview All Node Groups][all_node_groups_01]

As you can see, all **nodegroups** our assigned to the production environment.  Let's see what happens when we
override the environment. (using **/etc/puppetlabs/puppet/puppet.conf**, or on the command line):

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent --configprint environment
thias
[root@dl380 ~]# puppet agent -t --noop
Warning: Local environment: "thias" doesn't match server specified node environment "production", switching agent to "production".
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version '9d431ab93e69e47c851d1ba3b80138b465300b8e'
Notice: Finished catalog run in 0.77 seconds
[root@dl380 ~]# 
  {% endhighlight %}

The next experiment will be to change the environment of the **hypervisor** nodegroup to **thias**

Keep in mind that the environment you wish to couple to a node group must already exist.  Puppet Enterprise assembles the available
environments by reading the **environmentpath** from your **/etc/puppetlabs/puppet/puppet.conf**.

In our case, our environments are managed by **R10K**.

In the PE console, we can change the environment in the **edit node group metadata** of the **hypervisor** nodegroup page:

![Overview All Node Groups - thias environment][all_node_groups_02]

Let's see if the puppet run succeeds:

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop
Warning: Unable to fetch my node definition, but the agent run will continue:
Warning: Error 400 on SERVER: Classification failed for dl380.koewacht.net due to a classification conflict: The node was classified into groups named "hypervisor" and "PE MCollective" that defined conflicting values for the environment.
Info: Retrieving pluginfacts
Info: Retrieving plugin
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/application]/ensure: created
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/application/tp.rb]/ensure: defined content as '{md5}9ba96b22999374ddd71b36299cad62bd'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/commons]/ensure: created
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/commons/cache.rb]/ensure: defined content as '{md5}98dfe8b334e7e4b8d25de6d7803a0feb'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/face]/ensure: created
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/face/tp.rb]/ensure: defined content as '{md5}2152b16213e64c8bfebcbf117473d74d'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/parser/functions/bool2ensure.rb]/ensure: defined content as '{md5}59966c20db51d997fe65de89853046de'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/parser/functions/tp_content.rb]/ensure: defined content as '{md5}d603e77ba87d717eb51687c8a31804bf'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/parser/functions/tp_lookup.rb]/ensure: defined content as '{md5}9244edae8ded3093aaeaaba78fd97cef'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/parser/functions/tp_pick.rb]/ensure: defined content as '{md5}0310d496a69016fabaf65b1405c6c256'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/provider/.gitkeep]/ensure: defined content as '{md5}d41d8cd98f00b204e9800998ecf8427e'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/provider/libvirt_pool]/ensure: created
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/provider/libvirt_pool/virsh.rb]/ensure: defined content as '{md5}d57e4e4e1c31f2d2928215ade0ca443c'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/type/.gitkeep]/ensure: defined content as '{md5}d41d8cd98f00b204e9800998ecf8427e'
Notice: /File[/var/opt/lib/pe-puppet/lib/puppet/type/libvirt_pool.rb]/ensure: defined content as '{md5}ef0fd9980a376c674d9ea57bd344af5a'
Info: Loading facts
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Failed when searching for node dl380.koewacht.net: Classification failed for dl380.koewacht.net due to a classification conflict: The node was classified into groups named "hypervisor" and "PE MCollective" that defined conflicting values for the environment.
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
[root@dl380 ~]#
  {% endhighlight %}

This does not look good.

We see the pluginsync is looking at the __thias__ environment, and syncs the functions, types and
providers from the **libvirt** module to the node.
We have assigned two nodegroups, each having a different environment :

* default in production
* hypervisor in thias

And changing the environment of the **default** nodegroup is not an option.

We can solve this by enabling the **override all other environments** in the **edit node group metadata** screen, again
in the **hypervisor** nodegroup page.

![Enabling the Override Option][override_01]


![Override Environment Enabled][override_02]

Lets check what this brings us :

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version 'bc9d309914f3ed0a6785dabff74d263868e094a6'
Notice: Finished catalog run in 0.83 seconds
[root@dl380 ~]#
  {% endhighlight %}

That looks great. We have added a new environment, which is assigned to our hypervisor node.

Lets assign a class to it now. When you open the **node group** page, which you access using the
**classification** menu, you will see **Classes** (the third tab).  This is the place where you
assign classes to nodegroups.

![Add a Class to a Nodegroup][add_class]

When you click on the input field (see image above), Puppet Enterprise shows a drop down list of all the classes it has found
in the modulepath of the environment where the nodegroup belongs to.  In older versions, only the **production** environment was scanned.
Keep in mind that Puppet Enterprise rescans the environments by default every 30 minutes.
You can initiate a rescan by clicking on the **refresh** button, in the middle, right side of above image..

You can check the rescan timeout with following command :

  {% highlight shell-session %}
[root@pepuppet ~]# puppet master --configprint  environment_timeout
180
[root@pepuppet ~]#
  {% endhighlight %}

See the puppetlabs documentation about the [directory environment configuration][dir_env_conf], which explains this all in detail.

And in the next image, we have an overview of the class(es) added to a nodegroup.

![Assigned classes][class_added]

If we run puppet on our hypervisor, the **roles::hypervisor_single_node** should be assigned to the node.
Since I already assigned this class, nothing will happen during the puppet run, but we can verify the
assigned classes of the latest puppet run by looking at the **classes.txt** file. I never remember the
location of this file, but we can look it up with following command:

  {% highlight shell-session %}
[root@dl380 pe-puppet]# /opt/puppet/bin/puppet agent --configprint vardir
/var/opt/lib/pe-puppet
[root@dl380 pe-puppet]#
  {% endhighlight %}


  {% highlight shell-session %}
[root@dl380 pe-puppet]# grep roles::hypervisor_single_node /var/opt/lib/pe-puppet/classes.txt
roles::hypervisor_single_node
roles::hypervisor_single_node
[root@dl380 pe-puppet]#
  {% endhighlight %}

But do we now have back the full power of **R10K** ? 
Do we have the possibility to create new environments by creating new branches in our **master repo** ?
Can we overrule the environment on the agent, without the need to reassign nodegroups to those  environment over and over again ?

The answer is still **NO**.

Enabling The full power of R10K
------------------------------

Just keep on reading and we hope, at the end of this post, the **R10K** magic
will be working again.

To test our setup, I created a [dyn_env module][dyn_env] on github.

Now, let's create a new branch in our **master repo**, **development** :

  {% highlight shell-session %}
[witjoh@fc20 koewacht-newpuppet]$ cd pepuppet
[witjoh@fc20 pepuppet]$ git branch
* production
  thias
[witjoh@fc20 pepuppet]$ git branch development
[witjoh@fc20 pepuppet]$ git checkout development
Switched to branch 'development'
[witjoh@fc20 pepuppet]$ git branch
* development
  production
  thias
[witjoh@fc20 pepuppet]$
  {% endhighlight %}

next, we add our **dyn_env** module to the **Puppetfile** in the **development** branch. ( we are still working
in our **master repo**)

  {% highlight shell-session %}
[witjoh@fc20 pepuppet]$ git status
On branch development
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

  modified:   Puppetfile

no changes added to commit (use "git add" and/or "git commit -a")
[witjoh@fc20 pepuppet]$ git add Puppetfile
[witjoh@fc20 pepuppet]$ git status
On branch development
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

  modified:   Puppetfile

[witjoh@fc20 pepuppet]$ git commit -m "added dyn_env for testing"
[development 418d81e] added dyn_env for testing
 1 file changed, 3 insertions(+)
[witjoh@fc20 pepuppet]$ git push origin development
Counting objects: 7, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 337 bytes | 0 bytes/s, done.
Total 3 (delta 2), reused 0 (delta 0)
To https://github.com/witjoh/pepuppet.git
 * [new branch]      development -> development
[witjoh@fc20 pepuppet]$
  {% endhighlight %}

And run **R10K** on the puppet master :

  {% highlight shell-session %}
[root@pepuppet ~]# r10k deploy environment -pv
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/development
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/development/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/development/modules/dyn_env
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/production
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/production/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/thias
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/libvirt
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/roles
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/profiles
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/tp
  {% endhighlight %}

Did you see the magic ?? We do have a new environment, and the **dyn_env** module is now available.
But we are not there, yet :

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop --environment development
Warning: Local environment: "development" doesn't match server specified node environment "thias", switching agent to "thias".
  {% endhighlight %}

As you can see, we cannot switch to our new environment, because we are not allowed to override the environment.
In [this post][pe_user] you can find an explanation how to do proper **classification** to get the magic happen again.

Lets go back to the PE console.  I did some cleanup, as you can see in next image (I removed the hypervisor node group).

![Clean Classification][r10k_001]

We need to make sure the following :

* make it possible that my testnode can override its environment
* we can classify the module(s) under test

First thing we need to do is to create a nodegroup, with the **agent-specified environment** enabled, which I called **override**.

![New Nodegroup][r10k_002]

And we need also to enable the **override** feature

![Enable Override][r10k_002bis]

Next step is to add all the nodes that are allowed to override the 'PE console' assigned environments.
In this simple case, we pinned our testnode, instead of defining some rules.

![Pin Node][r10k_003]

And let's check if we can switch environments:

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop --environment thias
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version 'bc9d309914f3ed0a6785dabff74d263868e094a6'
Notice: Finished catalog run in 1.24 seconds
[root@dl380 ~]#
[root@dl380 ~]# puppet agent -t --noop --environment development
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version '418d81ee7fe4357833c637f9683eaac93cdd33aa'
Notice: Finished catalog run in 0.75 seconds
[root@dl380 ~]# puppet agent -t --noop --environment production
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version '9d431ab93e69e47c851d1ba3b80138b465300b8e'
Notice: Finished catalog run in 0.77 seconds
[root@dl380 ~]#
  {% endhighlight %}

Now we can switch environments as we like, if the environment exists on our puppet master,
otherwise we will get the following error

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop --environment faulty
Warning: Unable to fetch my node definition, but the agent run will continue:
Warning: Find /faulty/node/dl380.koewacht.net?transaction_uuid=30e5a44d-6e6e-4cdb-bda5-24ae0b6e321e&fail_on_404... resulted in 404 with the message: Not Found: Could not find environment 'faulty'
Info: Retrieving pluginfacts
Error: /File[/var/opt/lib/pe-puppet/facts.d]: Could not evaluate: Could not retrieve information from environment faulty source(s) puppet://pepuppet.koewacht.net/pluginfacts
Info: Retrieving plugin
    <snip .... >
Info: Loading facts
Error: Could not retrieve catalog from remote server: Find /faulty/catalog/dl380.koewacht.net?facts_format=pson&facts=%257B%2522name%2522%253A%2522dl380.koe... resulted in 404 with the message: Not Found: Could not find environment 'faulty'
Warning: Not using cache on failed catalog
Error: Could not retrieve catalog; skipping run
root@dl380 ~]# [root@dl380 ~]#
  {% endhighlight %}

It seems we have the behaviour we wanted.  Lets try to add some classes to this setup. We want to test our **dyn_env**,
and as you can see in, this class is not assigned to our node :

  {% highlight shell-session %}
[root@dl380 ~]# cat /var/opt/lib/pe-puppet/classes.txt
puppet_enterprise
puppet_enterprise::profile::mcollective::agent
settings
default
puppet_enterprise::params
puppet_enterprise
puppet_enterprise::profile::mcollective::agent
puppet_enterprise::mcollective::server
puppet_enterprise::mcollective::server::plugins
puppet_enterprise::mcollective::server::facter
puppet_enterprise::mcollective::server::logs
puppet_enterprise::mcollective::server::certs
puppet_enterprise::mcollective::service
[root@dl380 ~]#
  {% endhighlight %}

If we add the class in an intuitive way, by adding it to the **override** node group,
we encounter the following  error:

The first thing we notice, is that the drop down box is missing, and we have to type in the
class by hand.  As soon as it is in a way recognised, the **add class** button
becomes available.  But when we want to **commit 1 change**, we get following error.

> Error saving group: The group being edited or created makes reference to the following missing classes: "dyn_env" class in the "agent-specified" environment. See the `details` key for complete information on where each reference to a missing class originated.

![Commit Error][r10k_004]

We cannot assign classes to a nodegroup with the environment set to **agent-specified**.  We need to do this
somewhere else. In this case, we have a module which only exists in the **development branch**, and thus,
is only available in the **development environment**.

We created a **dummy** nodegroup, assigned it to the **development** environment, and added our **dyn_env** class to it.
As you can see in next picture, the class is indeed available in the drop down box.

![add dyn_env][r10k_005]

and now the commit does succeed :

![dyn_env added][r10k_006]

We are not there yet.
We still have to couple the **dyn_env** class to our node.

First, lets have a closer look at our testnode :

![node overview][r10k_007]

What do we see here :

* PE Mcollective classes
* we can override the environment (override group)

But we still have no **dyn_env** module assigned.

Since the **dyn_env** class is already added to the **dummy** nodegroup, we will
add this node to this group.  Again we will use the  **pin node**
feature.

![dyn_env assigned][r10k_008]

Now we should have our **module under test** assigned to the testnode.  Lets verify this in PE console:

![node overview after][r10k_009]

And when we run puppet on the testnode :

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop --environment development
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version '418d81ee7fe4357833c637f9683eaac93cdd33aa'
Notice: /Stage[main]/Dyn_env/Notify[Running in the development environment]/message: current_value absent, should be Running in the development environment (noop)
Notice: Class[Dyn_env]: Would have triggered 'refresh' from 1 events
Notice: Stage[main]: Would have triggered 'refresh' from 1 events
Notice: Finished catalog run in 0.77 seconds
[root@dl380 ~]#
  {% endhighlight %}

The final test - Adding a new environment
----------------------------------------

In the **dyn_env** class, we created a **feature** branch, which will have
an extra notify in the **dyn_env** class.  Check the [github repo][dyn_env] for the details.

In the **master repo**,we did the following actions :

* In the **development** branch
  * Pin the **dyn_env** module to the **master** branch in the **Puppetfile**
  * Push this to github
* Create a new branch in the **master repo**
  * Create the **testenv** branch
  * Checkout this branch
  * Pin the **dyn_env** module to its **feature** branch
  * Push this to github
* run **R10K** on the puppet master

  {% highlight shell-session %}
[root@pepuppet ~]# r10k deploy environment -pv
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/development
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/development/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/development/modules/dyn_env
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/production
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/production/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/testenv
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/testenv/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/testenv/modules/dyn_env
[R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/thias
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/libvirt
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/motd
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/roles
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/profiles
[R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/thias/modules/tp
[root@pepuppet ~]#
  {% endhighlight %}

Did you see the magic happen ?  We do have the **dyn_env** module now in two environments, **development** and **testenv**.

Lets do some puppet runs on my testnode :

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop --environment development
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version '76f0a70ea24c34e893a9c826b46e12e1ed66eb38'
Notice: /Stage[main]/Dyn_env/Notify[Running in the development environment]/message: current_value absent, should be Running in the development environment (noop)
Notice: Class[Dyn_env]: Would have triggered 'refresh' from 1 events
Notice: Stage[main]: Would have triggered 'refresh' from 1 events
Notice: Finished catalog run in 0.71 seconds
[root@dl380 ~]#
  {% endhighlight %}

  {% highlight shell-session %}
[root@dl380 ~]# puppet agent -t --noop --environment testenv
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Loading facts
Info: Caching catalog for dl380.koewacht.net
Info: Applying configuration version '0e6c29d178dc6917e499c209c4c7dfcba4b93a48'
Notice: /Stage[main]/Dyn_env/Notify[Running in the testenv environment]/message: current_value absent, should be Running in the testenv environment (noop)
Notice: /Stage[main]/Dyn_env/Notify[We are now in the feature branch]/message: current_value absent, should be We are now in the feature branch (noop)
Notice: Class[Dyn_env]: Would have triggered 'refresh' from 2 events
Notice: Stage[main]: Would have triggered 'refresh' from 1 events
Notice: Finished catalog run in 0.77 seconds
[root@dl380 ~]#
  {% endhighlight %}

Without changing something in the PE console, we can switch environments, and as you can see, we have different versions of the same
module in each environment.  We have again the **R10K** magic enabled.

---

Summary
=======

We did the following steps in the **PE Console** :

* Create one nodegroup, with the **agent-specified** environment assigned.
* Enable the **override** option on that group
* Add the nodes that are allowed to override the environment to this nodegroup

And for every class you want to assign to one of those nodes :

* Make the class visible for the **PE Console**
  * Add the class to an existing nodegroup
  * Make sure the class is available in the environment, only the class name matters, not the branch/version of is module
* Add the 'testnode' to a nodegroup having the class visible.

You should check in the console, in the node view, that all classes you want are visible.
To which environment they belong is not that important in this case.

If some class is not available in the **environment** you defined,  the catalog compilation will fail.
This is indeed the expected behaviour.

And modules, there versions, and environments, we manage with **R10K**.

Enjoy your **R10K** magic.

[thias-libvirt]: https://forge.puppetlabs.com/thias/libvirt
[dir_env_conf]: https://docs.puppetlabs.com/puppet/3.7/reference/environments_configuring.html
[dyn_env]: https://github.com/witjoh/dyn_env
[pe_user]: https://groups.google.com/a/puppetlabs.com/forum/#!topic/pe-users/k9oEsDEoSqc
[prev_post]: {% post_url 2014-12-26-Hypervisor-under-puppet_control %}


[PE - Grouping01]: https://docs.puppetlabs.com/pe/latest/console_classes_groups_getting_started.html
[PE - Grouping02]: https://docs.puppetlabs.com/pe/latest/console_classes_groups.html
[PE - Environments]: https://docs.puppetlabs.com/pe/latest/console_classes_groups_environment_override.html
[PE - Assigning]: https://docs.puppetlabs.com/pe/latest/puppet_assign_configurations.html

[all_node_groups_01]: {{ site.url }}/images/all_node_groups_overview.png
[all_node_groups_02]: {{ site.url }}/images/hypervisor_thias.png
[override_01]: {{ site.url }}/images/Enable_override.png
[override_02]: {{ site.url }}/images/override_enabled.png
[add_class]: {{ site.url }}/images/add_class.png
[class_added]: {{ site.url }}/images/class_added.png
[r10k_001]: {{ site.url }}/images/r10k_001.png
[r10k_002]: {{ site.url }}/images/r10k_002.png
[r10k_002bis]: {{ site.url }}/images/r10k_002bis.png
[r10k_003]: {{ site.url }}/images/r10k_003.png
[r10k_004]: {{ site.url }}/images/r10k_004.png
[r10k_005]: {{ site.url }}/images/r10k_005.png
[r10k_006]: {{ site.url }}/images/r10k_006.png
[r10k_007]: {{ site.url }}/images/r10k_007.png
[r10k_008]: {{ site.url }}/images/r10k_008.png
[r10k_009]: {{ site.url }}/images/r10k_009.png

