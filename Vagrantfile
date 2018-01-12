# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # All Vagrant configuration is done here. The most common configuration
  # options are documented and commented below. For a complete reference,
  # please see the online documentation at vagrantup.com.

  # Every Vagrant virtual environment requires a box to build off of.
  config.vm.box = "centos64"
  config.vm.box_url = "http://puppet-vagrant-boxes.puppetlabs.com/centos-64-x64-vbox4210.box"

  # The url from where the 'config.vm.box' box will be fetched if it
  # doesn't already exist on the user's system.
  # config.vm.box_url = "http://domain.com/path/to/above.box"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  config.vm.network "forwarded_port", guest: 8080, host: 8082
         
  # See http://stackoverflow.com/questions/11955525
  Vagrant::Config.run do |config|
    config.ssh.private_key_path = "~/.ssh/id_rsa"
    config.ssh.forward_agent = true
  end

  # See https://www.danpurdy.co.uk/web-development/osx-yosemite-port-forwarding-for-vagrant/
  # for Mac OS Yosemite 10.10
  config.trigger.after [:provision, :up, :reload] do
    system('echo "
        rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 8080 -> 127.0.0.1 port 8082
        rdr pass on lo0 inet proto tcp from any to 127.0.0.1 port 443 -> 127.0.0.1 port 4443
  " | sudo pfctl -ef - > /dev/null 2>&1; echo "==> Fowarding Ports: 8080 -> 8082 & Enabling pf"')
  end

  config.trigger.after [:halt, :destroy] do
    system("sudo pfctl -df /etc/pf.conf > /dev/null 2>&1; echo '==> Removing Port Forwarding & Disabling pf'")
  end
  
  config.vm.provision "shell", inline: <<-SHELL
   echo "\n----- Installing Java 8 ------\n"
   yum -y install java-1.8.0-openjdk
   echo "\n----- Installing Jenkins ------\n"
   wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
   rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
   yum -y install jenkins
   echo "\n----- Starting Jenkins ------\n"
   service jenkins start
   service jenkins status
   echo "\n----- firewall client ------\n"
   iptables -I INPUT -p tcp -m tcp --dport 8080 -j ACCEPT
   service iptables save
   echo "\n----- Now you will be able to access the guest's Jenkins at the address http://localhost:8082 ------\n"
   
   
  SHELL

end

