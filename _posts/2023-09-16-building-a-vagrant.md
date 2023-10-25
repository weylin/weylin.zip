---
layout: page
title:  "Building A Vagrant Env."
description: >
  These are just notes from a vagrant project I'm working on at the moment.
  Nothing about this really special or bloggish. 
date: 2023-09-27 15:55:57 -0400
hide_description: false
sitemap: false
---

### Resources

- https://tudip.com/blog-post/vagrant-setup/
- https://github.com/UtahDave/salt-vagrant-demo/blob/master/Vagrantfile
- https://tudip.com/blog-post/vagrant-setup/
- https://developer.hashicorp.com/vagrant/docs/provisioning/salt

### Steps & My Configs

1. Install virtualbox and vagrant on OS
2. In project build basic Vagrantfile (ruby) and install salt:

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

$script = <<-SCRIPT
echo I am provisioning...
date > /etc/vagrant_provisioned_at
wget -O bootstrap-salt.sh https://bootstrap.saltproject.io
sudo sh bootstrap-salt.sh
SCRIPT

Vagrant.configure("3") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.provision "shell", inline: $script
end
```

## Issues

I kept running into an error with key provisioning. I want a salt master, to mirror my current live environment, and in my case I want the master to have a minion; but during configurations was getting this error:

```
$ vagrant destroy master                                                 
==> bots: Forcing shutdown of VM...
==> bots: Destroying VM and associated drives...
There are errors in the configuration of this machine. Please fix
the following errors and try again:      

salt provisioner:                              
* You must accept keys when running highstate with master!

```

It was quite frustrating because the error doesn't clearly state what option(s) is missing. Looking around in the provisoiner code I found where the error is generated from:

https://github.com/hashicorp/vagrant/blob/a7135c000b55fbc2a346baf1dbf8169beedccf0f/plugins/provisioners/salt/config.rb#L163C1-L165

 The options all need to work in conjunction:  `if @install_master && !@no_minion && !@seed_master && @run_highstate`
```
   salt.install_master⋅=⋅true                                                
   salt.minion_config⋅=⋅"salt_minion/files/vagrant_minion.conf"              
   salt.run_highstate⋅=⋅true                                                 
   salt.seed_master⋅=⋅{          
     "salt-master" => "keys/master_minion.pem"                                            
	 "bots"⋅=>⋅"keys/bots.pem"                                               
   }   
```

You must pre-seed minion keys onto the master using the `seed_master` option, and include a dict of keys (instance names) with values (pem keys) of the path to the respective key.

In addition to that, since the master is running a local minion I need to give it it's own config to start up properly. Using the `salt.master_config` & `salt_minion_config` options and giving both paths to respective, basic, configs.

```
$ cat salt_minion/files/vagrant_minion.conf 
master: 192.168.56.10
hash: sha256
$ cat salt_master/files/vagrant_master.conf 
auto_accept: True
$ 
```
