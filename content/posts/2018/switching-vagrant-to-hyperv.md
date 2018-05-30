---
title: "Switching Vagrant to from VirtualBox to Hyper-V"
date: 2018-01-21T15:35:03+10:00
draft: false
tags: [Vagrant, VirtualBox, "Hyper-V"]
categories: [Software, Development]
---

I use Vagrant to script and automate virtual machines used for development
purposes. When I first started using Vagrant I didn't really understand its
utility or its place in the development cycle. Now though I'm quite happy in
my understanding that it is just a automation layer, a scripting abstraction,
used to paste over the differences in a collection of virtualization layers--
most of the time at least. 

## Just use something else!

Other alternative exist to using virtual machines for development
environments. Yet, I prefer VMs since:

- Just using Linux/Unix is not a desirable choice for me (for both professional
  and personal reasons)
- Using Bash for Windows (the Windows Subsystem for Linux) is not quite right
    - While a great general solution, up until recently has been only mostly
    feature complete
    - Additionally I like the idea of isolating my various projects and
    development environments to different VMs--this helps ensure effects of 
    conflicts remain isolated. The WSL is basically the same as installing all
    the things on my dev machine which is a mess I want to avoid.
- Docker is an option, though despite being keen to investigate Docker I don't
  have a need to use it yet.

VMs also have a reputation for using excessive resources. My answer to
that is to simply not to use GUIs; I find shell only access decreases required
resources considerably. This works for me as all of my Linux development
requires only shell access.


## The topic at hand: Vagrant providers

Vagrant can utilize different virtualization providers including
VirtualBox, Hyper-V, and others.
I had been using VirtualBox as my primary virtualization provider for many years
, but found it increasingly tricky to maintain. VirtualBox sometimes saw
infrequent updates, bugs, and required strange incantations to enable differing
functionality (see the note about symlinks below).
In addition, installing the Guest Additions on to every machine
was often frustrating. 

Given I had a brand new development machine, and it came with Hyper-V installed, I
decided to try and use Hyper-V instead. This has several benefits:

- Hyper-V and VirtualBox cannot be run on the same
  machine at the same time (there's a decent explanation
  [here](https://superuser.com/questions/1208850/why-vitualbox-or-vmware-can-not-run-with-hyper-v-enabled-windows-10#)).
  Picking one requires either disabling Hyper-V or not
  using VirtualBox. And editing boot entries to enable or disable Hyper-V is a
  non-starter for me.
- Hyper-V is (by my unfounded reasoning) a stable, performant, and reliable
  product as it is a production grade hypervisor. I paid for it in Windows, might
  as well take advantage of it.
- The recent Docker for Windows release relies on Hyper-V and it would be nice
  to be able to experiment easily with Docker in the future.

## 1. Enabling Hyper-V

For my machine it Hyper-V already enabled but if it is not for you, you can
[enable it](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v):

```PowerShell
Enable-WindowsOptionalFeature -Online -FeatureName:Microsoft-Hyper-V -All
```

## 2. Add a new provider to the Vagrant file

I'm adding a new provider (and not replacing the default provider) so that
other developers can still choose to use VirtualBox on their machines (including
those that develop on non-Windows machines).

The existing provider definition to use VirtualBox:

```ruby
  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant.
  config.vm.provider "virtualbox" do |vb|
    # Do not display the VirtualBox GUI when booting the machine
    vb.gui = false
    vb.name = "baw-deploy-vm"
    vb.memory = "1024"
    vb.cpus = 2
 
    # Enable symlink creation with Virtual Box's shared folders
    # http://stackoverflow.com/questions/24200333/symbolic-links-and-synced-folders-in-vagrant
    vb.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/vagrant", "1"]
  end
```

Next I added the Hyper-V definition. It was added before the VirtualBox provider
so that if Vagrant detects a valid Hyper-V install it will use that provider
first and fallback to the secondary VirtualBox definition if necessary.

```ruby
  config.vm.provider 'hyperv' do |hyperv|
    hyperv.vmname = 'baw-deploy-vm'
    # Customize the amount of memory on the VM:
    hyperv.memory = 1024
    hyperv.cpus = 2
  end
 ```

## 3. Configuring shared folders

Vagrant has various supported methods for sharing directories and files from
host to guest. This, for me, is the killer feature. I can use all my favourite
Windows tools (including IDEs) at native performance and still run only the code
that needs to run in the guest VM.
The trick, however, is to find a method that works (and more difficult:
one that reliably works) for the combination of host and guest operating systems.

- VirtualBox
    - No longer available for use since we aren't using the VirtualBox provider
- NFS
    - Is fast
    - Work natively for Linux guests/hosts
    - Needs a Vagrant plugin for Windows hosts
    - The [vagrant-winnfsd](https://github.com/winnfsd/vagrant-winnfsd) plugin
        works well _usually_, however there currently bugs with using it:
        - If the vagrant-winnfd plugin is even installed Vagrant doesn't start
          up properly! https://github.com/winnfsd/vagrant-winnfsd/issues/109
        - And https://github.com/hashicorp/vagrant/issues/7816
- RSync
    -  Works well on Linux
    - I can't find a stable, trusted, Win32 build
- SMB
    - Is the ideal solution as the SMB protocol is native to Windows
    - However I encountered spectacular difficulty using the method:
        - Some how Samba and Windows 10 disagree on the SMB protocol. There is some
            [bug](https://twitter.com/atruskie/status/951398733136056320) related to
            packet length that causes Samba to fail. It happens with Windows 10 build
            16299, and Ubuntu 14.x, 16.x, but not 17.10!
            {{< tweet 951398733136056320 >}}
        - Additionally, using a SMB higher than the v1.0 protocol (which is
            necessary due to vulnerabilities in the protocol), trigger a [bug in
            Vagrant](https://github.com/hashicorp/vagrant/issues/8620)
            which assumes which authentication protocol needs to be used
            {{< tweet 951426224584192000 >}}
- SSHFS (SSH File System)
    - Not one of Vagrant's inbuilt sharing options and requires a plugin to work:
      https://github.com/dustymabe/vagrant-sshfs
    - Thanks to the [Win32-OpenSSH](https://github.com/PowerShell/Win32-OpenSSH) project,
      a stable Windows implementation of SSH (and sftp) are now available to 
      Windows users, both allowing SSH host access and using SSH as a client
    - SSFS has been critiqued to be slow, but by disabling encryption (and
      caching) we can achieve decent performance.

As you may have surmised from the list, I would have chosen SMB or NFS if I
could have but due to bugs those options weren't viable. I went with the SSHFS
option and it seems to be working well.

First a helper to install the vagrant-sshfs plugin (since Vagrant files do not
support a formal notion of plugin dependencies):

```ruby
# http://matthewcooper.net/2015/01/15/automatically-installing-vagrant-plugin-dependencies/
required_plugins = %w( vagrant-sshfs )
required_plugins.each do |plugin|
  unless Vagrant.has_plugin? plugin || ARGV[0] == 'plugin'
    command_separator = Vagrant::Util::Platform.windows? ? " & " : ";"
    exec "vagrant plugin install #{plugin}#{command_separator}vagrant #{ARGV.join(" ")}"
  end
end
```

And then the shared folder definitions. Note, I previously had one set of folder
definitions, placed outside the provider definitions. Now I've added them to
each provider so we can use the best method for the provider:

```ruby
  # remove the default synced folder, since we map project dir elsewhere
  config.vm.synced_folder '.', '/vagrant', disabled: true
 
   config.vm.provider 'hyperv' do |hyperv|
     hyperv.vmname = 'baw-deploy-vm'
     hyperv.memory = 1024
     hyperv.cpus = 2
     
     # map project dir to home on guest, using sshfs. Turn off compression since
     # we are only sending data via loopback NIC. Also map file permissions to
     # the `vagrant` user and chmod all files to 600
     config.vm.synced_folder "./", "/home/vagrant/baw-deploy",
       type: "sshfs",
       ssh_opts_append: "-o Compression=no",
       sshfs_opts_append: "-o cache=no -o uid=1000 -o gid=1000 -o umask=0177"
   end
  
    config.vm.provider "virtualbox" do |vb|
      # Do not display the VirtualBox GUI when booting the machine
      vb.gui = false
      vb.name = "baw-deploy-vm"

      vb.memory = "1024"
      vb.cpus = 2
  
      # Enable symlink creation with Virtual Box's shared folders
      # http://stackoverflow.com/questions/24200333/symbolic-links-and-synced-folders-in-vagrant
      vb.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/vagrant", "1"]

      config.vm.synced_folder "./", "/home/vagrant/baw-deploy",
        mount_options: ["dmode=700,fmode=600"]
    end
```

We also need to make sure SSH key-forwarding is enabled:

```ruby
  # Enable ssh agent forwarding. Default is false.
  # This magic allows for the SSH key used to connect back to the host for the
  # SSH synced folders to be automatically sent to the guest!
  config.ssh.forward_agent = true
```

And that the private-key required to connect back to our host is available to be
forwarded:

```PowerShell
# host machine:
$ ssh-add ~/.ssh/host-machine-private-key
```

## Conclusion

And all of that worked--eventually. I'm pretty happy I got everything to work but
there definitely a lot of time-wasting bugs to be discovered along the way!

I've embedded a full copy of the Vagrant file below for reference:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# Install additional plugins
# http://matthewcooper.net/2015/01/15/automatically-installing-vagrant-plugin-dependencies/
required_plugins = %w( vagrant-sshfs )
required_plugins.each do |plugin|
  unless Vagrant.has_plugin? plugin || ARGV[0] == 'plugin'
    command_separator = Vagrant::Util::Platform.windows? ? " & " : ";"
    exec "vagrant plugin install #{plugin}#{command_separator}vagrant #{ARGV.join(" ")}"
  end
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Enable ssh agent forwarding. Default is false.
  # This magic allows for the SSH key used to connect back to the host for the
  # SSH synced folders to be automatically sent to the guest!
  config.ssh.forward_agent = true

  # Every Vagrant development environment requires a box.
  config.vm.box = "bento/ubuntu-14.04"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  # config.vm.network "private_network", ip: "192.168.33.10"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  config.vm.network "public_network"

  # remove the default synced folder, since we map project dir elsewhere
  config.vm.synced_folder '.', '/vagrant', disabled: true
 
  config.vm.provider 'hyperv' do |hyperv|
    hyperv.vmname = 'baw-deploy-vm'
    hyperv.memory = 1024
    hyperv.cpus = 2
    
    # map project dir to home on guest, using sshfs. Turn off compression since
    # we are only sending data via loopback NIC. Also map file permissions to
    # the `vagrant` user and chmod all files to 600
    config.vm.synced_folder "./", "/home/vagrant/baw-deploy",
      type: "sshfs",
      ssh_opts_append: "-o Compression=no",
      sshfs_opts_append: "-o cache=no -o uid=1000 -o gid=1000 -o umask=0177"
  end

  config.vm.provider "virtualbox" do |vb|
    # Do not display the VirtualBox GUI when booting the machine
    vb.gui = false
    vb.name = "baw-deploy-vm" 
    vb.memory = "1024"
    vb.cpus = 2
 
    # Enable symlink creation with Virtual Box's shared folders
    # http://stackoverflow.com/questions/24200333/symbolic-links-and-synced-folders-in-vagrant
    vb.customize ["setextradata", :id, "VBoxInternal2/SharedFoldersEnableSymlinksCreate/vagrant", "1"] 
    config.vm.synced_folder "./", "/home/vagrant/baw-deploy",
      mount_options: ["dmode=700,fmode=600"]
  end
  
  # Enable provisioning with a shell script.
  config.vm.provision 'shell', path: "./setup-control-node.sh"
end  
```