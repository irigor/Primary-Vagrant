# Note: much of the documentation and code in this file is from Varying Vagrant Vagrants, the original base for this project

vagrant_dir = File.expand_path(File.dirname(__FILE__))

Vagrant.configure("2") do |config|

	# Store the current version of Vagrant for use in conditionals when dealing
	# with possible backward compatible issues.
	vagrant_version = Vagrant::VERSION.sub(/^v/, '')

	# Default Ubuntu Box
	#
	# This box is provided directly by Bento and is updated regularly.
	config.vm.box = "bento/ubuntu-16.04"

	config.vm.hostname = "pv"

	# Default Box IP Address
	#
	# This is the IP address that your host will communicate to the guest through. In the
	# case of the default `192.168.56.101` that we've provided, Virtualbox will setup another
	# network adapter on your host machine with the IP `192.168.13.1` as a gateway.
	#
	# If you are already on a network using the 192.168.13.x subnet, this should be changed.
	# If you are running more than one VM through Virtualbox, different subnets should be used
	# for those as well. This includes other Vagrant boxes.
	config.vm.network :private_network, id: "pv_primary", ip: "192.168.13.101"

	# Local Machine Hosts
	#
	# If the Vagrant plugin Ghost (https://github.com/10up/vagrant-ghost) is
	# installed, the following will automatically configure your local machine's hosts file to
	# be aware of the domains specified below. Watch the provisioning script as you may need to
	# enter a password for Vagrant to access your hosts file.
	#
	# By default, we'll include the domains set up by Primary Vagrant through the pv-hosts file
	# located in the default-sites/ directory.
	#
	# Other domains can be automatically added by including a pv-hosts file containing
	# individual domains separated by whitespace in subdirectories of user-data/ and user-date/sites/.
	if defined?(VagrantPlugins::Ghost)
		# Recursively fetch the paths to all pv-hosts files under the default-sites/, user-data/ and user-data/sites/ directories.
		paths = Dir[File.join(vagrant_dir, 'default-sites', 'pv-hosts')] + Dir[File.join(vagrant_dir, 'user-data', 'pv-hosts')]+ Dir[File.join(vagrant_dir, 'user-data', 'sites', '**', 'pv-hosts')]

		# Parse the found pv-hosts files for host names.
		hosts = paths.map do |path|

		# Read line from file and remove line breaks
		lines = File.readlines(path).map(&:chomp)

		# Filter out comments starting with "#"
		lines.grep(/\A[^#]/)

		end.flatten.uniq # Remove duplicate entries

		# Pass the found host names to the ghost plugin so it can perform magic.
		config.ghost.hosts = hosts
	end

	# Forward Agent
	#
	# Enable agent forwarding on vagrant ssh commands. This allows you to use identities
	# established on the host machine inside the guest. See the manual for ssh-add
	config.ssh.forward_agent = true

	# Configurations from 1.0.x can be placed in Vagrant 1.1.x specs like the following.
	config.vm.provider :virtualbox do |v|
		v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
		v.customize ["modifyvm", :id, "--memory", 1024]
		v.customize ["modifyvm", :id, "--name", "Primary Vagrant"]
		v.customize ["modifyvm", :id, "--cpus", 1]
		v.customize ["modifyvm", :id, "--natdnsproxy1", "on"]

		# set auto_update to false, if you do NOT want to check the correct
		# additions version when booting this machine
		config.vbguest.auto_update = true
	end

	# Don't check for updates with every vagrant up
	config.vm.box_check_update = false

	# Drive mapping
	#
	# The following config.vm.share_folder settings will map directories in your Vagrant
	# virtual machine to directories on your local machine. Once these are mapped, any
	# changes made to the files in these directories will affect both the local and virtual
	# machine versions. Think of it as two different ways to access the same file. When the
	# virtual machine is destroyed with `vagrant destroy`, your files will remain in your local
	# environment.

	# Custom Mappings
	#
	# Use this to insert your own (and possibly rewrite) Vagrant config lines. Helpful
	# for mapping additional drives. If a file 'pv-mappings' exists in the user-data/ folder or user-data/sites or any of its subfolders
	# it will be evaluated as ruby inline as it loads.
	if File.exists?(File.join(vagrant_dir,'user-data', 'pv-mappings')) then
		eval(IO.read(File.join(vagrant_dir, 'user-data', 'pv-mappings')), binding)
	end
	Dir[File.join( vagrant_dir, 'user-data', 'sites', '**', 'pv-mappings')].each do |file|
    	eval(IO.read(file), binding)
    end
    eval(IO.read(File.join(vagrant_dir, 'default-sites', 'pv-mappings')), binding)

	# Provisioning
	#
	# Process one or more provisioning scripts depending on the existence of custom files.
	config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'" # Prevents stdin error for Ubuntu

    # Make sure puppet is on the box
	config.vm.provision "shell",
        inline: "apt-get update && apt-get install -y software-properties-common puppet"

    # Run the system setup on the first vagrant up.
	config.vm.provision "puppet" do |puppet|
		puppet.manifests_path    = "provision"
		puppet.manifest_file     = "init"
		puppet.module_path       = "provision/modules"
		puppet.facter            = { "fqdn" => "pv" }
		puppet.hiera_config_path = "provision/hiera.yaml"
	end

    # Run provisioning required every time we startup the box.
    config.vm.provision "puppet", run: "always" do |puppet|
        puppet.manifests_path    = "user-data/"
        puppet.manifest_file     = "vhosts"
        puppet.module_path       = "provision/modules"
        puppet.facter            = { "fqdn" => "pv" }
        puppet.hiera_config_path = "provision/hiera.yaml"
    end

	# Vagrant Triggers
    #
    # If the vagrant-triggers plugin is installed, we can run various scripts on Vagrant
    # state changes like `vagrant up`, `vagrant halt`, `vagrant suspend`, and `vagrant destroy`
    #
    # These scripts are run on the host machine, so we use `vagrant ssh` to tunnel back
    # into the VM and execute things. By default, each of these scripts calls db_backup
    # to create backups of all current databases. This can be overridden with custom
    # scripting. See the individual files in provision/lib/bin for details.
    if defined? VagrantPlugins::Triggers
        config.trigger.before :up do
            system('./provision/bin/repo_init.sh')
            if File.exists?(File.join(vagrant_dir,'user-data', 'pv-init.sh')) then
                system('./user-data/pv-init.sh')
            end
            Dir[File.join( 'user-data/sites', '**', 'pv-init.sh')].each do |file|
                print file
                system(file)
            end
        end
        config.trigger.before :halt do
            run_remote "bash /vagrant/provision/lib/bin/vagrant_halt"
            if File.exists?(File.join(vagrant_dir,'user-data', 'pv-halt.sh')) then
                system('./user-data/pv-halt.sh')
            end
            Dir[File.join( 'user-data/sites', '**', 'pv-halt.sh')].each do |file|
                print file
                system(file)
            end
        end
        config.trigger.before :suspend do
            run_remote "bash /vagrant/provision/lib/bin/vagrant_suspend"
            if File.exists?(File.join(vagrant_dir,'user-data', 'pv-suspend.sh')) then
                system('./user-data/pv-suspend.sh')
            end
            Dir[File.join( 'user-data/sites', '**', 'pv-suspend.sh')].each do |file|
                print file
                system(file)
            end
        end
        config.trigger.before :destroy do
            run_remote "bash /vagrant/provision/lib/bin/vagrant_destroy"
            if File.exists?(File.join(vagrant_dir,'user-data', 'pv-destroy.sh')) then
                system('./user-data/pv-destroy.sh')
            end
            Dir[File.join( 'user-data/sites', '**', 'pv-destroy.sh')].each do |file|
                print file
                system(file)
            end
        end
    end
end
