# -*- mode: ruby -*-
# # vi: set ft=ruby :

# For help on using kubespray with vagrant, check out docs/developers/vagrant.md

require 'fileutils'
require 'ipaddr'
require 'socket'

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), ENV['MOLECULE_VAGRANT_CONFIG'] || 'vagrant/config.rb')

# Uniq disk UUID
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "rockylinux9"         => {box: "rockylinux/9",               user: "vagrant"}
}

if File.exist?(CONFIG)
  require CONFIG
end

# Defaults for config options defined in CONFIG
$num_instances ||= 3
$instance_name_prefix ||= "kafka"
$vm_gui ||= false
$vm_memory ||= 1024
$vm_cpus ||= 1
$shared_folders ||= {}
$forwarded_ports ||= {}
$subnet ||= "192.168.60"
$os ||= "rockylinux9"
$inventories ||= []
# Modify those to have separate groups (for instance, to test separate kafka_controller:)
# first_kafka_broker = 1
# first_kafka_controller = 4
# kafka_broker_instances = 3
# kafka_controller_instances = 3
$first_node ||= 1
$first_kafka_broker ||= 1
$first_kafka_controller ||= 1

# The first three nodes are kafka brokers
$kafka_broker_instances ||= [$num_instances, 3].min
# The second three nodes are kafka controllers
$kafka_controller_instances ||= [$num_instances, 3].min
# All nodes are kafka nodes
$kafka_node_instances ||= $num_instances - $first_node + 1

$override_disk_size ||= false
$disk_size ||= 20480
$local_path_provisioner_enabled ||= "False"
$local_path_provisioner_claim_root ||= "/opt/local-path-provisioner/"
# boolean or string (e.g. "-vvv")
$ansible_verbosity ||= false
$ansible_tags ||= ENV['VAGRANT_ANSIBLE_TAGS'] || ""

$vagrant_dir ||= File.join(File.dirname(__FILE__), ".vagrant")

$playbook ||= "cluster.yml"
$extra_vars ||= {}

host_vars = {}

def collect_networks(subnet)
  ip_test = IPAddr.new("#{subnet}.0")
  Socket.getifaddrs.each_with_object([]) do |iface, acc|
    addr    = iface.addr
    netmask = iface.netmask
    next unless addr && netmask && addr.afamily == Socket::AF_INET

    begin
      ip     = IPAddr.new(addr.ip_address)
      prefix = IPAddr.new(netmask.ip_address).to_i.to_s(2).count('1')
      network = IPAddr.new("#{ip.mask(prefix)}/#{prefix}")
      acc << [network, ip_test]
    rescue ArgumentError
      next
    end
  end
end

def subnet_in_use?(network_ips)
  network_ips.any? { |net, test_ip| net.include?(test_ip) }
end

network_ips = collect_networks($subnet)

if subnet_in_use?(network_ips)
  puts "Invalid subnet provided, subnet is already in use: #{$subnet}.0"
  puts "Subnets in use: #{network_ips.inspect}"
  exit 1
end

# throw error if os is not supported
if ! SUPPORTED_OS.key?($os)
  puts "Unsupported OS: #{$os}"
  puts "Supported OS are: #{SUPPORTED_OS.keys.join(', ')}"
  exit 1
end

$box = SUPPORTED_OS[$os][:box]

if Vagrant.has_plugin?("vagrant-proxyconf")
  $no_proxy = ENV['NO_PROXY'] || ENV['no_proxy'] || "127.0.0.1,localhost"
  (1..$num_instances).each do |i|
      $no_proxy += ",#{$subnet}.#{i+100}"
  end
end

Vagrant.configure("2") do |config|

  config.vm.box = $box
  if SUPPORTED_OS[$os].has_key? :box_url
    config.vm.box_url = SUPPORTED_OS[$os][:box_url]
  end
  config.ssh.username = SUPPORTED_OS[$os][:user]

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  # always use Vagrants insecure key
  config.ssh.insert_key = false

  if ($override_disk_size)
    unless Vagrant.has_plugin?("vagrant-disksize")
      system "vagrant plugin install vagrant-disksize"
    end
    config.disksize.size = $disk_size
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "%s-%01d" % [$instance_name_prefix, i] do |node|

      node.vm.hostname = vm_name

      if Vagrant.has_plugin?("vagrant-proxyconf")
        node.proxy.http     = ENV['HTTP_PROXY'] || ENV['http_proxy'] || ""
        node.proxy.https    = ENV['HTTPS_PROXY'] || ENV['https_proxy'] ||  ""
        node.proxy.no_proxy = $no_proxy
      end

      node.vm.provider :virtualbox do |vb|
        vb.memory = $vm_memory
        vb.cpus = $vm_cpus
        vb.gui = $vm_gui
        vb.linked_clone = true
        vb.customize ["modifyvm", :id, "--vram", "8"]
        vb.customize ["modifyvm", :id, "--audio", "none"]
        vb.customize ['storagectl', :id, '--name', 'SATA Controller', '--add', 'sata', '--controller', 'IntelAhci', '--portcount', '8']
      end

      if $expose_docker_tcp
        node.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      $forwarded_ports.each do |guest, host|
        node.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
      end

      if ["rhel8"].include? $os
        # Vagrant synced_folder rsync options cannot be used for RHEL boxes as Rsync package cannot
        # be installed until the host is registered with a valid Red Hat support subscription
        node.vm.synced_folder ".", "/vagrant", disabled: false
        $shared_folders.each do |src, dst|
          node.vm.synced_folder src, dst
        end
      else
        node.vm.synced_folder ".", "/vagrant", disabled: false, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z'] , rsync__exclude: ['.git','venv']
        $shared_folders.each do |src, dst|
          node.vm.synced_folder src, dst, type: "rsync", rsync__args: ['--verbose', '--archive', '--delete', '-z']
        end
      end

      ip = "#{$subnet}.#{i+100}"
      node.vm.network :private_network,
        :ip => ip

      # Disable swap for each vm
      node.vm.provision "shell", inline: "swapoff -a"


      # Disable firewalld on redhat based vms
      if ["rhel8","rockylinux8","rhel9","rockylinux9"].include? $os
        node.vm.provision "shell", inline: "systemctl stop firewalld; systemctl disable firewalld"
      end

      host_vars[vm_name] = {
        "ip": ip,
        "download_localhost": "False",
        "local_path_provisioner_enabled": "#{$local_path_provisioner_enabled}",
        "local_path_provisioner_claim_root": "#{$local_path_provisioner_claim_root}",
        "ansible_ssh_user": SUPPORTED_OS[$os][:user],
        "ansible_ssh_private_key_file": File.join(Dir.home, ".vagrant.d", "insecure_private_key"),
        "unsafe_show_logs": "True"
      }

      # Only execute the Ansible provisioner once, when all the machines are up and ready.
      # And limit the action to gathering facts, the full playbook is going to be ran by testcases_run.sh
      if i == $num_instances
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = $playbook
          ansible.compatibility_mode = "2.0"
          ansible.verbose = $ansible_verbosity
          ansible.become = true
          ansible.limit = "all,localhost"
          ansible.host_key_checking = false
          ansible.raw_arguments = ["--forks=#{$num_instances}",
                                   "--flush-cache",
                                   "-e ansible_become_pass=vagrant"] +
                                   $inventories.map {|inv| ["-i", inv]}.flatten
          ansible.host_vars = host_vars
          ansible.extra_vars = $extra_vars
          if $ansible_tags != ""
            ansible.tags = [$ansible_tags]
          end
          ansible.groups = {
            "kafka_broker" => ["#{$instance_name_prefix}-[#{$first_kafka_broker}:#{$kafka_broker_instances + $first_kafka_broker - 1}]"],
            "kafka_controller" => ["#{$instance_name_prefix}-[#{$first_kafka_controller}:#{$kafka_controller_instances + $first_kafka_controller - 1}]"],
            "kafka_node" => ["#{$instance_name_prefix}-[#{$first_node}:#{$kafka_node_instances + $first_node - 1}]"],
            "kafka_cluster:children" => ["kafka_broker", "kafka_node"],
          }
        end
      end

    end
  end
end
