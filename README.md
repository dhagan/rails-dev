# rails-dev
https://gorails.com/guides/using-vagrant-for-rails-development

Overview
 This will take about 20 minutes.

Vagrant is a tool to automatically setup a development environment inside a virtual machine on your computer. This means your development environment can exactly match the production server and your coworkers can all run exactly the same software.

Your Rails development environment will stay the same no matter what your development computer setup is like. Plus, when you need to revisit a project 12 months later, you can get up and running within minutes. Future you will thank you for setting it up at the beginning of your project.

We're also going to use Chef which helps us automate how our virtual machine development environment gets setup. It will take care of setting up Ruby and all the other packages on our system. It's pretty rad.

Enough of a sales pitch, let's get to it.

Setting Up Vagrant
Make sure you have 1GB or 2GB of free RAM on your computer before proceeding because Vagrant runs a full operating system inside a virtual machine to run your Rails applications.

The first step is to install Vagrant and VirtualBox on your computer.

Install Vagrant
Install VirtualBox

Install vagrant - http://www.vagrantup.com/downloads.html
Install Virtual Box - https://www.virtualbox.org/wiki/Downloads


VirtualBox is where your virtual machines will run. They will be headless which means that they will run in the background and you will interact with them over SSH.

Next we're going to install two plugins for Vagrant.

vagrant-vbguest automatically installs the host's VirtualBox Guest Additions on the guest system.
vagrant-librarian-chef let's us automatically run chef when we fire up our machine.
vagrant plugin install vagrant-vbguest
vagrant plugin install vagrant-librarian-chef-nochef
This can take a while.

Create the Vagrant config
First off, hop into a Rails project that you want to setup Chef for and run the following commands.

cd MY_RAILS_PROJECT # Change this to your Rails project directory
vagrant init
touch Cheffile
This will create both the Vagrantfile and Cheffile for us to customize.

Your Cheffile
Now we're going to setup our Cheffile. This file is just like your Rails Gemfile but for Chef. This file defines the Chef cookbooks that we will use in our project. Later on in the Vagrantfile we will tell Vagrant how to use these cookbooks to setup our environment.

We just want to paste the following code into our Cheffile:

site "http://community.opscode.com/api/v1"

cookbook 'apt'
cookbook 'build-essential'
cookbook 'mysql', '5.5.3'
cookbook 'ruby_build'
cookbook 'nodejs'
cookbook 'rbenv', git: 'https://github.com/aminin/chef-rbenv'
cookbook 'vim'
Your Vagrantfile
Our Vagrantfile defines the operating system and Chef configuration for our virtual machine.

We're going to be using Ubuntu 14.04 trusty 64-bit (change to "trusty32" if you need to use 32-bit) with 2GB of memory. It is also going to forward port 3000 from the virtual machine to our computer so when we run rails server we can access the server inside the virtual machine from our regular browser. Last but not least, we have Chef setup Ruby 2.2.1 and MySQL inside our VM.

You can replace the Vagrantfile contents with the following:

# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Use Ubuntu 14.04 Trusty Tahr 64-bit as our operating system
  config.vm.box = "ubuntu/trusty64"

  # Configurate the virtual machine to use 2GB of RAM
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "2048"]
  end

  # Forward the Rails server default port to the host
  config.vm.network :forwarded_port, guest: 3000, host: 3000

  # Use Chef Solo to provision our virtual machine
  config.vm.provision :chef_solo do |chef|
    chef.cookbooks_path = ["cookbooks", "site-cookbooks"]

    chef.add_recipe "apt"
    chef.add_recipe "nodejs"
    chef.add_recipe "ruby_build"
    chef.add_recipe "rbenv::user"
    chef.add_recipe "rbenv::vagrant"
    chef.add_recipe "vim"
    chef.add_recipe "mysql::server"
    chef.add_recipe "mysql::client"

    # Install Ruby 2.2.1 and Bundler
    # Set an empty root password for MySQL to make things simple
    chef.json = {
      rbenv: {
        user_installs: [{
          user: 'vagrant',
          rubies: ["2.2.1"],
          global: "2.2.1",
          gems: {
            "2.2.1" => [
              { name: "bundler" }
            ]
          }
        }]
      },
      mysql: {
        server_root_password: ''
      }
    }
  end
end
Running Vagrant
Now that we've got Vagrant and Chef configured properly, we'll start up the Vagrant virtual machine and ssh into it.

# The commented lines are the output you should see when you run these commands

vagrant up
#==> default: Checking if box 'ubuntu/trusty64' is up to date...
#==> default: Clearing any previously set forwarded ports...
#==> default: Installing Chef cookbooks with Librarian-Chef...
#==> default: The cookbook path '/Users/chris/code/test_app/site-cookbooks' doesn't exist. Ignoring...
#==> default: Clearing any previously set network interfaces...
#==> default: Preparing network interfaces based on configuration...
#    default: Adapter 1: nat
#==> default: Forwarding ports...
#    default: 3000 => 3000 (adapter 1)
#    default: 22 => 2222 (adapter 1)
#==> default: Running 'pre-boot' VM customizations...
#==> default: Booting VM...
#==> default: Waiting for machine to boot. This may take a few minutes...
#    default: SSH address: 127.0.0.1:2222
#    default: SSH username: vagrant
#    default: SSH auth method: private key
#    default: Warning: Connection timeout. Retrying...
#==> default: Machine booted and ready!
#==> default: Checking for guest additions in VM...
#==> default: Mounting shared folders...
#    default: /vagrant => /Users/chris/code/test_app
#   default: /tmp/vagrant-chef-1/chef-solo-1/cookbooks => /Users/chris/code/test_app/cookbooks
#==> default: VM already provisioned. Run `vagrant provision` or use `--provision` to force it

vagrant ssh
#Welcome to Ubuntu 14.04 LTS (GNU/Linux 3.13.0-24-generic x86_64)
#
# * Documentation:  https://help.ubuntu.com/
#
# System information disabled due to load higher than 1.0
#
#  Get cloud support with Ubuntu Advantage Cloud Guest:
#    http://www.ubuntu.com/business/services/cloud
#
#
#vagrant@vagrant-ubuntu-trusty-64:~$
The first time you run vagrant up will take a while because it will provision your virtual machine with the chef configuration. After the first time, vagrant up won't have to run Chef and it will boot much faster.

If you ever edit your Vagrantfile or Cheffile, you can use the following command to reconfigure the machine.

vagrant provision
Using Rails inside Vagrant
Vagrant sets up the /vagrant folder as a shared directory between the virtual machine and your host operating system. If you cd /vagrant and run ls you will see all the files from your Rails application.

bundle to install all your gems inside the virtual machine.

rbenv rehash to make sure the executables from the gems we just installed (like rails) are available.

rake db:create && rake db:migrate to create and migrate your database.

rails server from this directory will run your Rails application on port 3000. You will still be able to access Rails just like usual on localhost:3000.


