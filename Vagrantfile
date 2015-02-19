# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

unless Vagrant.has_plugin?("vagrant-aws")
   puts "-- WARNING --"
   puts "This Vagrantfile makes use of the vagrant-aws plugin."
   puts "This package is necessary to interact with aws."
   puts " "
   puts "execute: \"vagrant plugin install vagrant-aws\" to continue"

   exit
end


CONFIG = File.join(File.dirname(__FILE__), "config.rb")

#what provider are we executing for?
provider_is_aws  = (!ARGV.nil? && ARGV.join('').include?('provider=aws'))
provider_is_vmware = (!ARGV.nil? && ARGV.join('').include?('provider=vmware'))
provider_is_virtualbox = (!ARGV.nil? && ARGV.join('').include?('provider=virtualbox'))
provider_is_google = (!ARGV.nil? && ARGV.join('').include?('provider=google'))


# Defaults :q# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end

if File.exist?(CONFIG)
  require CONFIG
end

Vagrant.configure("2") do |config|

config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= 308.0.1"
  config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel

  config.vm.provider :vmware_fusion do |vb, override|
    override.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant_vmware_fusion.json" % $update_channel
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

 ["google"].each do |google|
	config.vm.provider google do |z, override|
        override.vm.box = "gce"
	override.vm.box_url = "https://github.com/mitchellh/vagrant-google/raw/master/google.box"
	override.vm.box_version = ""
	end
     end

 ["aws"].each do |aws|
     config.vm.provider aws do |x, override|
     override.vm.box = "dummy"
     override.vm.box_url = "https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box"
     override.vm.box_version = ""
    end
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "core-%02d" % i do |config|
      config.vm.hostname = vm_name

      if config.vm.hostname == "core-01"
	if provider_is_vmware or provider_is_google
	theuserdata = File.read("user-data.weavemaster.vmware")
        cloud_config_path = File.join(File.dirname(__FILE__), "user-data.weavemaster.vmware")
	else
	theuserdata = File.read("user-data.weavemaster")
	cloud_config_path = File.join(File.dirname(__FILE__), "user-data.weavemaster")
	end
	else
        if provider_is_vmware or provider_is_google
	theuserdata =  File.read("user-data.vmware")
	cloud_config_path = File.join(File.dirname(__FILE__), "user-data.vmware")
	else
	theuserdata = File.read("user-data")
	cloud_config_path = File.join(File.dirname(__FILE__), "user-data")
	end
    end

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        config.vm.provider :vmware_fusion do |v, override|
          v.vmx["serial0.present"] = "TRUE"
          v.vmx["serial0.fileType"] = "file"
          v.vmx["serial0.fileName"] = serialFile
          v.vmx["serial0.tryNoRxLoss"] = "FALSE"
        end

        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      config.vm.provider :vmware_fusion do |vb|
        vb.gui = $vb_gui
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = $vb_gui
        vb.memory = $vb_memory
        vb.cpus = $vb_cpus
      end

      ip = "172.17.8.#{i+100}"
      config.vm.network :private_network, ip: ip

      # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
      #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']
      config.vm.synced_folder '.', '/vagrant', disabled: true

     ##do the goog setup
     ["google"].each do |google|
	config.vm.provider google do |g, override|
		g.google_project_id = ENV['GC_PROJECT']
		g.google_client_email = ENV['GC_CLIENT_EMAIL']
		g.google_key_location = ENV['GC_KEY_LOCATION']
		g.machine_type = ENV['GC_MACHINETYPE']
		g.image = ENV['GC_IMAGE']
		g.name = vm_name
		g.metadata = {'user-data' => theuserdata }
		override.ssh.username = "core"
		override.ssh.private_key_path = "~/.ssh/google_compute_engine"
	#	override.ssh.private_key_path = "~/.ssh/id_rsa"
		end
	    end

     ##do the aws setup
     ["aws"].each do |aws|
        config.vm.provider aws do |a, override|
                a.access_key_id = ENV['AWS_KEY']
                a.secret_access_key = ENV['AWS_SECRET']
                a.keypair_name = ENV['AWS_KEYNAME']
		a.ami = ENV['AWS_AMI'] 
		a.region = ENV['AWS_REGION'] 
		a.instance_type = ENV['AWS_INSTANCE']
		a.security_groups =  ENV['AWS_SECURITYGROUP']
		a.ami = ENV['AWS_AMI']
		a.user_data = theuserdata
                override.ssh.private_key_path = ENV['AWS_KEYPATH']
                override.ssh.username = "core"
           end
	end

	unless provider_is_aws or provider_is_google
        config.vm.provision :file, :source => "#{cloud_config_path}", :destination => "/tmp/vagrantfile-user-data"
        config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
	end

    end
  end
end
