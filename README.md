
# Introduction

In networking research we sometimes need to use Virtual Machines (VMs) to achieve a specific goal, e.g., creating a topology and emulating multiple nodes with a single host, separation of Host/Guest operating system, for example when developing kernel a module, etc.
VMs also make it easy to share code along with its dependencies and to ease code reproducibility and longevity.
This document covers the approach to VM management that we found to work best along with notes for some of the components.

## Management Software

To manage VMs we use [Vagrant](https://www.vagrantup.com/).
Vagrant is a tool that automates VM creation, manage, and deletion. 
From the Vagrant docs:
> Vagrant isolates dependencies and their configuration within a single disposable, consistent environment, without sacrificing any of your existing tools.
To achieve this Vagrant uses Vagrantfiles.
A Vagrantfile contains all the creation, provisioning, and setup instructions for the VM.

You can find out more about Vagrant, how to install it, and manage VMs [here](https://developer.hashicorp.com/vagrant/tutorials/getting-started).


## Hypervisors

To create a VM Vagrant needs a hypervisor. 
A hypervisor is a piece of software that allows virtualization of resources and enables running virtual machines on a physical machine.

My prefered hypervisor is [Qemu](https://www.qemu.org/).
Qemu requires the [libvirt](https://libvirt.org/) provider to work with Vagrant.
The vagrant tutorial specifies how to setup and install [Virtualbox](https://www.virtualbox.org/) as a hypervisor.
The following explains my motivation for using qemu over Virtualbox.


### Using Qemu over Virtualbox
    
For my specific usecase, using mininet within a VM to create a network topology, I found that using Virtualbox (6.1.x) creates latency variation within the mininet nodes.
On the other hand, qemu with libvirt has shown consistent, expected, network latency with little to no deviation within the mininet nodes.
As my work relies on accurate network RTT samples, I chose to use quemu over Virtualbox.

I also found that sets up the VM-Host networking differently.
Virtualbox sets up ssh access to the host machine by port forwarding port 22 on the VM to a port on the host.
Qemu sets up a virtual network, assigns a private IP to the VM and uses that address and port 22 for secure shell access.
This is a small quality of life feature which makes it easier to connect multiple VMs on one host without having to deal with port colision or manually dealing with port forwarding to instruct which host port should be used for which VM.

### Setting up Qemu and Libvirt with Vagrant

The following [link](https://opensource.com/article/21/10/vagrant-libvirt) aids how to install libvirt and qemu on Debian- and RedHat-based distributions.
For MacOS, both qemu and libvirt are available to install through homebrew.

The vagrant-libvirt plugin is needed to start qemu and libvirt.
To install the vagrant-libvirt you need to use ``vagrant plugin install vagrant-libvirt``.

# Sample Vagrantfile    

A sample [Vagrantfile](https://github.com/glasgow-ipl/vagrant-notes/blob/master/Vagrantfile) is provided.
It sets up an Ubuntu 20.04 VM from the generic hashicorp repository.
The machine is provisioned with 8 cores and 16GB RAM, these parameters are configurable.
This VM does not come with any desktop or graphics packages. 
These need to be installed separately if required.
The sample Vagrantfile also has a section which shows how X11 forwarding can be [enabled](https://github.com/glasgow-ipl/vagrant-notes/blob/master/Vagrantfile#L13), 
though use of X11 is not recommended as it can be very slow over the network.

The provisioning script is ran once only when the VM is created.
This can be useful to setup the VM packages, pull all dependencies, compile the tool that will be used for experiments later, etc.
An example of a provisioning script is [provided](https://github.com/glasgow-ipl/vagrant-notes/blob/master/Vagrantfile#L24).

## Using Multiple Providers

If you have more than one vagrant provider you may need to specify the provider when issuing using up, e.g., ``vagrant up --provider=libvirt``.
Ryo found that you can add the following [line](https://github.com/glasgow-ipl/vagrant-notes/blob/master/Vagrantfile#L21) to the vagrantfile to force use of a specific driver and provider (e.g., qemu and libvert).
``libvirt.driver`` option should be set to ``kvm``.

## Sharing config in multi-provider environment

You can write Vagrantfile to target multiple providers. Whichever provider available/defaulted will be used without selecting explicitly as shown in above section. 

Vagrantfile is just a ruby script; you can set up your own variable, e.g. `common_cpu=8` and use it in the respective provider configs. 

``Vagrantfile-multi_config_example`` shows an example where virtualbox provider and libvirt provider is both set-up. 

## Vagrant+libvirt environment VM name-collision
In a shared-machine context, vagrant+libvirt setup can have an issue where the vms overwrite each other. The libvirt VM machine name defaults to just the directory name containing the Vagrantfile and suffix ``-default``. 
vagrant-libvirt plugin has a way to set the prefix (https://github.com/vagrant-libvirt/vagrant-libvirt/issues/289) via ``libvirt.default_prefix`` variable. 
The Vagrantfile examples in this repository is configured to use the unix UID, random 10 digit number, and the containing directory as the prefix. 

## Shared Directories and moving data between the VM and the Host
    
The provided sample Vagrantfile sets up a shared directory `/vagrant` at the VM and the directory containing the Vagrantfile `.` at the host.
Using shared folders we encountered **many issues** with synchronisation, host vs guest CPU clocks, 
permissions, and others.
The provided share folder uses NFS (version 4) to synchronise the files and forces use of TCP.
This has been the most reliable method we have found to work.
Setting host file file permission mask may also be required to ensure that files have the correct permissions.

In general, we discourage the use of shared folders and suggest that all data generated by the VM is kept in the VM while data generation occurs (e.g., single instance of an experiment run).
After that the data may be transferred to the host, for example, using rsync.
Using rsync is also easier to automate when qemu is used because of the network setup that qemu uses by default.

One way to use rsync with vagrant is to dump the ssh config onto a file and feed that file to ssh command as an option for rsync command (or any other program that uses ssh internally)

Running `vagrant ssh-config > .ssh_config` in the same directory as Vagrantfile will create a file called .ssh_config in the directory.
When running `rsync`, `-e` option can be used to feed ssh command and arguments to establish connection with the VM. e.g.:
```
rsync -avH -e "ssh -F ./.ssh_config" default:~/results/ ./results/" 
```
Above will synchronise ~/results within the vagrant vm and the ./results folder right next to the Vagrantfile. 

When destroying the vagrant vm, remember to delete the .ssh_config to avoid confusion and to prompt re-generating the .ssh_config file again.


# Known issue
    Last updated: 2025-04-23
    There seems to be some issues with bento/Ubuntu 22.04 and libvirt. See [github issue] (https://github.com/chef/bento/issues/1604) on this. Suggest to use (generic/Ubuntu 22.04 image)[https://portal.cloud.hashicorp.com/vagrant/discover/generic/ubuntu2204] instead.


#

This document is ever-evolving. Feel free to add your own contributions, clarify the points made, or insert additional content.
