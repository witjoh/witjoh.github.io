---
layout: post
title:  "Setting up r10k on Puppet Enterprise"
date:   2014-12-26 10:12:00
categories: Puppet
tags:
  - Puppet Enterprise
  - r10k
  - github
  - environments
---
Setting up Dynamic Environments with Puppet Enterprise and github
=================================================================

This setup is heavily based on the famous [Shit Gary Says][garys-r10k] blogs about environments and r10k.

But first, a short summary of the state of my new setup @home :

* dl380 CentOS7 acting as hypervisor
* pepuppet VM freshly installed

Since I do not have a local git server yet, I will be using one of my github accounts to put everything under version control.
That is something we really want, right ? , Version control, and from the beginning. We will be setting up our own git server later on.

We also will stick with the default settings of the modules and manifest directories, which on puppet enterprise are :

  {% highlight bash %}
     [root@pepuppet environments]# puppet master --configprint modulepath
     /etc/puppetlabs/puppet/environments/production/modules:/etc/puppetlabs/puppet/modules:/opt/puppet/share/puppet/modules
     [root@pepuppet environments]# puppet master --configprint manifest
     /etc/puppetlabs/puppet/environments/production/manifests
  {% endhighlight %}

Keep in mind that these are the settings for the 'production' environment, which is the default one when omitting the --environment parameter.
In Puppet Enterprise, the production environment is already setup, with a minimal configuration :

* The manifests/site.pp is created with minimal settings
* an empty module directory is also created

Also, PE is already configured correctly for the 'Directory Environments' setup.

Adjusting my github to work better with r10k
============================================

The only thing I wish to do here is to rename gits default branch 'master' to 'production'.  This way, we are matching git's branches to puppet
environments.
Well, I will have to setup a new repository first on github, which will hold all needed files for the r10k setup.

This is one way to rename my 'master' branch to 'production' :

* git clone https://github.com/witjoh/pepuppet.git
* cd pepuppet/
* git branch -m master production
* git push origin production
* Go to GitHub UI settings/options, and change the 'Default Branch' to 'production'
* git push origin :master


Now we only have a production branch.

Installing r10k
===============

(just following Gary's blog)

* puppet module install zack/r10k
* modify Gary's configure_10rk.pp manifest (point to my github repository, and adjusted the r10k version to 1.4.0)

  {% highlight puppet %}
    [root@pepuppet ~]# cat configure_r10k.pp
    ######         ######
    ##  Configure R10k ##
    ######         ######

    ##  This manifest requires the zack/R10k module and will attempt to
    ##  configure R10k according to my blog post on directory environments.
    ##  Beware! (and good luck!)

    class { 'r10k':
      version           =e-r10k-setup.markdown2014-12-26-Puppet_enterprise-r10k-setup.markdown '1.4.0',
      sources           => {
        'puppet' => {
          'remote'  => 'https://github.com/witjoh/pepuppet.git',
          'basedir' => "${::settings::confdir}/environments",
          'prefix'  => false,
        }
      },
      purgedirs         => ["${::settings::confdir}/environments"],
      manage_modulepath => false,
    }
  {% endhighlight %}

* Transfer the file to my pepuppet master
* puppet apply configure_r10k.pp

  {% highlight bash %}
    Notice: Compiled catalog for pepuppet.koewacht.net in environment production in 0.65 seconds
    Notice: /Stage[main]/Git/Package[git]/ensure: created
    Notice: /Stage[main]/R10k::Install/Package[r10k]/ensure: created
    Notice: /Stage[main]/R10k::Install::Pe_gem/File[/usr/bin/r10k]/ensure: created
    Notice: /Stage[main]/R10k::Config/File[r10k.yaml]/ensure: defined content as '{md5}b143dc96d92c3d560a1e67962836dd53'
    Notice: Finished catalog run in 30.29 seconds
  {% endhighlight %}

Create initial files in the Control repo
========================================
In our case, this will be an empty Puppetfile, and starting with all the other needed files from Gary's examples.
Whenever the needs arise, those files will be adjusted when the need arises.

  {% highlight bash %}
    [witjoh@fc20 pepuppet]$ git status
    On branch production

    Changes to be committed:

      new file:   Puppetfile
      new file:   environment.conf
      new file:   hiera.yaml
      new file:   hieradata/common.yaml
      new file:   hieradata/dev.yaml
      new file:   hieradata/prod.yaml
      new file:   hieradata/test.yaml
      new file:   manifests/site.pp

    [witjoh@fc20 pepuppet]$ git commit -m "initial version, based on Gareth's great blog posts on workflows"
    [witjoh@fc20 .git]$ git push
  {% endhighlight %}


Time to see if r10k works.  Looking at the modules installed in our production environment, we have the following list :

  {% highlight bash %}
    [root@pepuppet manifests]# puppet module list
    /etc/puppetlabs/puppet/environments/production/modules
    ├── gentoo-portage (v2.2.0)
    ├── mhuffnagle-make (v0.0.2)
    ├── puppetlabs-concat (v1.1.2)
    ├── puppetlabs-gcc (v0.2.0)
    ├── puppetlabs-git (v0.3.0)
    ├── puppetlabs-inifile (v1.2.0)
    ├── puppetlabs-pe_gem (v0.1.0)
    ├── puppetlabs-ruby (v0.4.0)
    ├── puppetlabs-stdlib (v4.5.0)
    ├── puppetlabs-vcsrepo (v1.2.0)
    └── zack-r10k (v2.5.1)
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
    [root@pepuppet manifests]# 
  {% endhighlight %}

We did install the zack-r10k modules, which pulled in a lot of other modules due to its dependencies.
In our Puppetfile, we have not defined any module yet, so let's see what r10k is doing with my puppet server :

  {% highlight bash %}
    [root@pepuppet ~]# r10k deploy environment -pv
    [R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/production
    [R10K::Action::Deploy::Environment - ERROR] Command exited with non-zero exit code:
    Command: git clone --reference /var/cache/r10k/https---github.com-witjoh-pepuppet.git https://github.com/witjoh/pepuppet.git /etc/puppetlabs/puppet/environments/production
    Stderr:
    fatal: destination path '/etc/puppetlabs/puppet/environments/production' already exists and is not an empty directory.
    Exit code: 128
    [root@pepuppet ~]# 
  {% endhighlight %}

Oops !!!! This is a git error, it won't clone a repository to an existing directory, so we have to interfere manually, by moving our current production environment tree out of the way :

  {% highlight bash %}
    [root@pepuppet ~]# mv /etc/puppetlabs/puppet/environments/production /etc/puppetlabs/puppet/environments/production_old
    [root@pepuppet ~]# r10k deploy environment -pv
  {% endhighlight %}

which will give me the following result :

  {% highlight bash %}
    [root@pepuppet ~]# tree /etc/puppetlabs/puppet/environments/
    /etc/puppetlabs/puppet/environments/
    `-- production
        |-- environment.conf
        |-- hieradata
        |   |-- common.yaml
        |   |-- dev.yaml
        |   |-- prod.yaml
        |   `-- test.yaml
        |-- hiera.yaml
        |-- manifests
        |   `-- site.pp
        |-- Puppetfile
        `-- README.md

    3 directories, 9 files
    [root@pepuppet ~]#
  {% endhighlight %}

Ooops !!! And yes, because we have enabled purging in the r10k.yaml file (purgedirs option), we even get rid of the
production tree I moved aside.

I already have one module on github, so I added it to the Puppetfile, pushed it to github and reran r10k :

  {% highlight bash %}
    [root@pepuppet ~]# r10k deploy environment -pv
    [R10K::Action::Deploy::Environment - INFO] Deploying environment /etc/puppetlabs/puppet/environments/production
    [R10K::Action::Deploy::Environment - INFO] Deploying module /etc/puppetlabs/puppet/environments/production/modules/motd
    [root@pepuppet ~]#
  {% endhighlight %}

Yeah- it works.

Moving to the next step, but that will be another post.











[garys-r10k]: http://garylarizza.com/blog/2014/08/31/r10k-plus-directory-environments/
