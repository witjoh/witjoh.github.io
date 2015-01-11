---
layout: post
title:  "Classifying classes to a nodegroup with agent-specified environment set"
date:   2015-01-11 23:05:00
categories: puppet
tags:
  - Puppet Enterprise
  - R10K
  - Dynamic Environments

---
Classifying classes to a nodegroup with agent-specified environment set
=======================================================================

These are a bunch of screenshots I took trying to make classification work.
Still looking to PE 3.7.1, r10k and dynamic environments work properly together.
I hope to add apossible solution shortly.

The involved node groups
------------------------

![Overeview all node groups][Add_class_groups_01.png]
![Details thias testing group][Add_class_groups_03.png]
![Details hypervisor group][Add_class_groups_02.png]


Adding a class to the *hypervisor* node group
---------------------------------------------
![Adding a class to the node group][Add_class00.png]

As soon as the GUI finds the module in the modulepath, the *add class* button comes active.  Motd is only in the thias environment, thats why I created the *thias testing* node group with environment *thias*.

![Class is added, waiting commit][Add_class01.png]

![Error message commit][Add_class02.png]

    Error saving group: The group being edited or created makes reference to the
    following missing classes: "motd" class in the "agent-specified" environment.
    See the `details` key for complete information on where each reference to a
    missing class originated.


![When trying to leave the screen][Add_class03.png]


As a sidenode, when adding a *node.pp* with the node declaration in the environments manifest directory, classification does work.

More info will follow as soon as this is worked out properly

[Add_class_groups_01.png]: {{ site.url }}/images/Add_class_groups_01.png "Overeview all node groups"
[Add_class_groups_02.png]: {{ site.url }}/images/Add_class_groups_03.png "Details hypervisor group"
[Add_class_groups_03.png]: {{ site.url }}/images/Add_class_groups_03.png "Details thias testing group"
[Add_class00.png]: {{ site.url }}/images/Add_class00.png "Adding a class to the node group"
[Add_class01.png]: {{ site.url }}/images/Add_class01.png "Class is added, waiting commit"
[Add_class02.png]: {{ site.url }}/images/Add_class02.png "Error message commit"
[Add_class03.png]: {{ site.url }}/images/Add_class03.png "When trying to leave the screen"

