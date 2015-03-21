title: "Multi-Machine Setup with Vagrant and Saltstack"
date: 2014-11-14 08:50:00
categories: 
  - Configuration Management
  - Virtualisierung
tags:
  - vagrant
  - saltstack
  - docker
  - vm
---

[Vagrant](http://vagrantup.com) is a great way for developing and testing your
configurations before deploying to real hosts. In combination with the
[Saltstack](http://www.saltstack.com/) provisioner, you can run your salt
states on virtual machines and achieve really fast iteration cycles. While the
Vagrant docs give a
[great example for masterless quickstart](https://docs.vagrantup.com/v2/provisioning/salt.html)
with Vagrant and Saltstack, they do not elaborate on multi-machine
configurations, like you are likely to encounter in bigger setups.

In this post I will present you my two-machine Vagrantfile, which can be used
as a starting point for your individual setup. My example configures a master
host running salt-master and one minion, which will be provisioned for running
[Docker](http://docker.com).

<!-- more -->

## tl;dr

You can find the repository for this example
[on Github](https://github.com/mbrgm/vagrant-saltstack-example).

First, clone the repository:

```bash
$ git clone https://github.com/mbrgm/vagrant-saltstack-example.git
```

You need the
[vagrant-hostmanager](https://github.com/smdahlen/vagrant-hostmanager) and 
[vagrant-cachier](https://github.com/fgrehm/vagrant-cachier) plugins, so
install them:

```bash
$ vagrant plugin install vagrant-hostmanager vagrant-cachier
```

Then, bring the vagrant hosts up from inside your working copy:

```bash
$ cd vagrant-saltstack-example
$ vagrant up
```

You can now ssh into the master and run the salt highstate:

```bash
$ vagrant ssh master1 -c "sudo salt '*' state.highstate"
```
For simplicity, I also added a target running that command in a Makefile:

```bash
$ make update
```

## Detailed description

This section explains the Vagrantfile in greater detail. First, I'll describe
the individual sections. You can find the complete file at the end of the post.

```ruby

  #===================
  # Package cache
  #===================

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
```

I strongly recommend you use the
[vagrant-cachier](https://github.com/fgrehm/vagrant-cachier) plugin for your
Vagrantfiles. It shares a common package cache among similar VM instances and
has support for multiple package managers and Linux distros. We set the scope
to the 'box' level, which means that machines running from the same base box
share a common cache.


```ruby
  #================
  # Networking
  #================

  config.vm.network :private_network, type: "dhcp"

  config.hostmanager.enabled = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
    if vm.id
      `VBoxManage guestproperty get #{vm.id} "/VirtualBox/GuestInfo/Net/1/V4/IP"`.split()[1]
    end
  end
  config.vm.provision :hostmanager
```

We configure our machines to use an additional private network for
communication. Getting IPs via DHCP saves us the trouble of configuring them
manually for each machine. Then we enable the hostmanager plugin. The
hostmanager manages the /etc/hosts file of each machine, so that machines can
lookup each other's IP using the Vagrant hostnames and optional aliases. This
enables us to specify a 'salt' alias for the salt-master host, so the minions
can connect to the master without any additional configuration. The custom IP
resolver is a workaround to make hostmanager work with DHCP addresses. Once
[smdahlen/vagrant-hostmanager#86](https://github.com/smdahlen/vagrant-hostmanager/issues/86)
is resolved, this should be obsolete. In the last line, we tell vagrant to use
the hostmanager provisioner.

```ruby
  #==============
  # Machines
  #==============
  
  # Set array of hostnames
  minions = [ 'minion1' ]

  #-----------------
  # Salt master
  #-----------------

  config.vm.define "master1" do |node|

    node.vm.box      = "ubuntu-14.10-server-64"
    node.vm.hostname = "master1"
    node.hostmanager.aliases = %w(salt)

    node.vm.synced_folder "./salt/roots/salt", "/srv/salt"

    node.vm.provision :salt do |salt|

      salt.install_master = true

      master_keys_hash = { "master1" => "./salt/keys/master1/minion.pub" }
      minion_keys_hash = Hash[minions.map{
        |hostname| [hostname, "./salt/keys/#{hostname}/minion.pub"]
      }]

      salt.seed_master = master_keys_hash.merge(minion_keys_hash)

      salt.minion_key = "./salt/keys/master1/minion.pem"
      salt.minion_pub = "./salt/keys/master1/minion.pub"

    end

  end
```

In order to easily support multiple minions, we define a list of hostnames,
which will later be mapped to according minion configurations.  Next, we define
our salt master. As I wrote before, the hostmanager plugin can add additional
aliases for a host. That way minions will be able to reach the salt-master
under the 'salt' hostname. The salt root directory is shared using a synced
folder. If you want an additional pillar root, you must add another synced
folder. `salt.install_master = true` tells the salt provisioner that this
machine is going to be a a salt-master (the salt-minion is installed by
default).

The Vagrant Saltstack provisioner offers a way for preseeding the master keys,
so you don't have to ssh into the master and accept them manually. The hostname
=> pubkey map is created dynamically using the list of minion hostnames. I
included keys for one master and one minion in the example repo. If you
want to create your own keys or increase the number of minions, you can run
`make genkeys` from the repository root (provided you have openssl
installed). This task takes the configured machines' hostnames and creates
a public/private key pair for each one.

In order to make the preseeded keys work, we also have to configure the minions
to use the generated keys via the `salt.minion_key` and `salt.minion_pub`
options. As the master host also runs a salt-minion daemon, we have to tell it
the location of its keypair as well.


```ruby
  #------------------
  # Salt minions
  #------------------

  # Create a machine for each hostname
  minions.each do |hostname|
    config.vm.define hostname do |node|

      node.vm.box      = "ubuntu-14.10-server-64"
      node.vm.hostname = hostname 

      node.vm.provision :salt do |salt|
        salt.minion_key = "./salt/keys/#{hostname}/minion.pem"
        salt.minion_pub = "./salt/keys/#{hostname}/minion.pub"
      end

    end
  end
```

The last section of the Vagrantfile creates a machine running salt-minion for
every hostname in the `minions` list. There is nothing special about the minion
configuration. As with the master, the minions must use the generated keys for
the preseeding to work.

You can now bring the configured machines up and start your salt development:

```bash
$ vagrant up

# ... wait for machines to be provisioned

$ vagrant ssh master1 -c "sudo salt '*' test.ping"
master1:
    True
minion1:
    True
Connection to 127.0.0.1 closed.
```

The salt state tree in the example repo configures minion1 to be a docker host.
I also included a make task for quickly running `state.highstate` during
development, so a simple

```bash
$ make update
```

should get your docker host up and running. Note that no states have been
specified for the master1 minion, so salt will give you a warning about that.


And, at last, the complete Vagrantfile. Have fun!

```ruby Vagrantfile https://github.com/mbrgm/vagrant-saltstack-example/blob/73e644e865cfa3989c657769b734e4496103b235/Vagrantfile View on GitHub
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  #===================
  # Package cache
  #===================

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end

  
  #================
  # Networking
  #================

  config.vm.network :private_network, type: "dhcp"

  config.hostmanager.enabled = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
    if vm.id
      `VBoxManage guestproperty get #{vm.id} "/VirtualBox/GuestInfo/Net/1/V4/IP"`.split()[1]
    end
  end
  config.vm.provision :hostmanager

  
  #==============
  # Machines
  #==============
  
  # Set array of hostnames
  minions = [ 'minion1' ]

  #-----------------
  # Salt master
  #-----------------

  config.vm.define "master1" do |node|

    node.vm.box      = "ubuntu-14.10-server-64"
    node.vm.hostname = "master1"
    node.hostmanager.aliases = %w(salt)

    node.vm.synced_folder "./salt/roots/salt", "/srv/salt"

    node.vm.provision :salt do |salt|

      salt.install_master = true

      master_keys_hash = { "master1" => "./salt/keys/master1/minion.pub" }
      minion_keys_hash = Hash[minions.map{
        |hostname| [hostname, "./salt/keys/#{hostname}/minion.pub"]
      }]

      salt.seed_master = master_keys_hash.merge(minion_keys_hash)

      salt.minion_key = "./salt/keys/master1/minion.pem"
      salt.minion_pub = "./salt/keys/master1/minion.pub"

    end

  end

  
  #------------------
  # Salt minions
  #------------------

  # Create a machine for each hostname
  minions.each do |hostname|
    config.vm.define hostname do |node|

      node.vm.box      = "ubuntu-14.10-server-64"
      node.vm.hostname = hostname 

      node.vm.provision :salt do |salt|
        salt.minion_key = "./salt/keys/#{hostname}/minion.pem"
        salt.minion_pub = "./salt/keys/#{hostname}/minion.pub"
      end

    end
  end

end
```
