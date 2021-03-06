Vagrant.configure('2') do |config|
  config.vm.box     = 'dummy'
  config.vm.box_url = 'https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box'

  config.omnibus.chef_version = :latest

  # Workaround for "sudo: sorry, you must have a tty to run sudo" error. See
  # https://github.com/mitchellh/vagrant/issues/1482 for details.
  config.ssh.pty = true

  config.vm.provider :aws do |aws, override|
    # Workaround for "~/aws/keys/#{aws.region}/#{ENV['USER']}.pem", which for
    # some reason expands to an object instead of a string. E.g. the following
    # fails:
    #
    #   override.ssh.private_key_path = ENV['VAGRANT_SSH_PRIVATE_KEY_PATH'] ||
    #                                   "~/aws/keys/#{aws.region}/#{ENV['USER']}.pem"
    #
    aws_region            = ENV['VAGRANT_AWS_REGION']        || 'eu-west-1'
    aws.region            = aws_region

    aws.keypair_name      = ENV['VAGRANT_AWS_KEYPAIR_NAME']  || ENV['USER']
    aws.instance_type     = ENV['VAGRANT_AWS_INSTANCE_TYPE'] || 'm1.medium'

    # See http://aws.amazon.com/ec2/instance-types/#instance-details the
    # instance types that support this.
    aws.ebs_optimized     = false

    # TODO Auto-create the 'MongoDB' security group, or at least document
    # manual steps. Inbound ports on 22 (SSH) and 27017 (MongoDB) should be
    # allowed.
    aws.security_groups = [ 'MongoDB' ]

    # List of Amazon Linux AMIs (e.g. amzn-ami-pv-2013.09.0.x86_64-ebs).
    # Add the one for your region if it is missing.
    aws.region_config 'ap-southeast-1', :ami => 'ami-14f2b946'
    aws.region_config 'eu-west-1',      :ami => 'ami-149f7863'
    aws.region_config 'us-east-1',      :ami => 'ami-35792c5c'

    override.ssh.username         = 'ec2-user'
    override.ssh.private_key_path = ENV['VAGRANT_SSH_PRIVATE_KEY_PATH'] ||
                                    "~/aws/keys/#{aws_region}/#{ENV['USER']}.pem"
  end

  # See http://docs.mongodb.org/manual/administration/production-notes/ for
  # details.
  config.vm.provision :chef_solo do |chef|
    chef.add_recipe 'utils'
    chef.add_recipe 'mosh'
    chef.add_recipe 'ebs'
    chef.add_recipe 'mongodb::10gen_repo'
    chef.add_recipe 'mongodb'

    chef.cookbooks_path = ['cookbooks', 'my_cookbooks']

    if ENV['VAGRANT_DEBUG']
      chef.log_level = :debug
    end

    chef.json = {
      :mongodb => {
        :dbpath     => '/data',
        :smallfiles => true      # Speed up initial journal preallocation
      },
      :ebs => {
        :access_key         => ENV['AWS_ACCESS_KEY'],
        :secret_key         => ENV['AWS_SECRET_KEY'],
        :fstype             => 'ext4',  # Or 'xfs'
        :no_boot_config     => true,
        :md_read_ahead      => 32,      # Size in number of 512B sectors (16KB)
        # :mdadm_chunk_size => 256,
      }
    }

    if ENV['VAGRANT_EBS_RAID']
      chef.json[:ebs][:raids] = {
        '/dev/md0' => {
          :num_disks     => 4,
          :disk_size     => 10,    # Size in GB
          :raid_level    => 10,
          :fstype        => chef.json[:ebs][:fstype],
          :mount_point   => '/data',
          :mount_options => 'noatime,noexec',
          # :piops       => 2000,  # Provisioned IOPS
          # :uselvm      => true
        }
      }
    else
      chef.json[:ebs][:volumes] = {
        '/data' => {
          :size          => 20,  # Size in GB
          :fstype        => chef.json[:ebs][:fstype],
          :mount_options => 'noatime,noexec'
        }
      }
    end
  end
end
