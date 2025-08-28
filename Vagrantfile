# -*- mode: ruby -*-
# vi: set ft=ruby :

# For help on using kubespray with vagrant, check out docs/developers/vagrant.md

require 'fileutils'
require 'ipaddr'
require 'socket'

Vagrant.require_version ">= 2.0.0"

CONFIG = File.join(File.dirname(__FILE__), ENV['MOLECULE_VAGRANT_CONFIG'] || 'vagrant/config.rb')

# Uniq disk UUID
DISK_UUID = Time.now.utc.to_i

SUPPORTED_OS = {
  "rockylinux9" => { box: "rockylinux/9", user: "vagrant" }
}

if File.exist?(CONFIG)
  require CONFIG
end

# -----------------------------
# Defaults for config options
# -----------------------------
# Separate prefixes for roles
$broker_instance_name_prefix     ||= "kafka_broker_node"
$controller_instance_name_prefix ||= "kafka_controller_node"

$vm_gui                ||= false
$vm_memory             ||= 1024
$vm_cpus               ||= 1
$forwarded_ports       ||= {}
$subnet                ||= "192.168.60"
$os                    ||= "rockylinux9"

$first_node                 ||= 1
$kafka_broker_instances     ||= 3
$kafka_controller_instances ||= 3
$first_kafka_broker         ||= 1
$first_kafka_controller     ||= 4

# All nodes (brokers + controllers)
$kafka_node_instances ||= $kafka_broker_instances + $kafka_controller_instances - $first_node + 1

$override_disk_size   ||= false
$disk_size            ||= 20480
$ansible_verbosity    ||= false
$ansible_tags         ||= ENV['VAGRANT_ANSIBLE_TAGS'] || ""

$vagrant_dir ||= File.join(File.dirname(__FILE__), ".vagrant")

$playbook    ||= "tests/playbooks/kafka_cluster.yml"
$inventory   ||= "tests/inventories/testing"
$extra_vars  ||= {}

# -----------------------------
# Helpers
# -----------------------------
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

def should_check_subnet?
  cmd = (ARGV[0] || "").to_s
  return false if ENV["VAGRANT_NO_SUBNET_CHECK"] == "1"
  %w[up reload resume].include?(cmd)
end

# -----------------------------
# Subnet collision check
# -----------------------------
if should_check_subnet?
  network_ips = collect_networks($subnet)
  if subnet_in_use?(network_ips)
    msg = <<~MSG
      Invalid subnet: #{$subnet}.0/24 is already in use on the host.
      Conflicting networks detected: #{network_ips.map { |net, _| net.to_s }.uniq.join(', ')}
      (Set VAGRANT_NO_SUBNET_CHECK=1 to bypass this check — not recommended.)
    MSG
    abort msg  # prints the message to STDERR and exits with code 1
  end
end

# -----------------------------
# OS check
# -----------------------------
unless SUPPORTED_OS.key?($os)
  puts "Unsupported OS: #{$os}"
  puts "Supported OS are: #{SUPPORTED_OS.keys.join(', ')}"
  exit 1
end

$box = SUPPORTED_OS[$os][:box]

# -----------------------------
# Vagrant config
# -----------------------------
Vagrant.configure("2") do |config|
  config.vm.box = $box
  config.vm.box_check_update = false
  if SUPPORTED_OS[$os].key?(:box_url)
    config.vm.box_url = SUPPORTED_OS[$os][:box_url]
  end
  config.ssh.username = SUPPORTED_OS[$os][:user]

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  # always use Vagrant's insecure key
  config.ssh.insert_key = false

  if $override_disk_size
    unless Vagrant.has_plugin?("vagrant-disksize")
      system "vagrant plugin install vagrant-disksize"
    end
    config.disksize.size = $disk_size
  end

  (1..$kafka_node_instances).each do |i|
    is_broker = i.between?($first_kafka_broker, $first_kafka_broker + $kafka_broker_instances - 1)
    is_controller   = i.between?($first_kafka_controller, $first_kafka_controller + $kafka_controller_instances -    1)

    raise "Index #{i} not in broker/controller ranges" unless is_broker || is_controller

    role_idx =
      if is_broker
        i - $first_kafka_broker + 1   # 1..$kafka_broker_instances
      else
        i - $first_kafka_controller + 1  # 1..$kafka_controller_instances
      end

prefix  = is_broker ? $broker_instance_name_prefix : $controller_instance_name_prefix
vm_name = "#{prefix}_#{role_idx}"   # e.g., kafka_broker_node_1, kafka_controller_node_1

config.vm.define vm_name do |node|
    host_name = vm_name.tr("_", "-")
    node.vm.hostname = host_name

      node.vm.provider :virtualbox do |vb|
        vb.check_guest_additions = false
        vb.memory                = $vm_memory
        vb.cpus                  = $vm_cpus
        vb.gui                   = $vm_gui
        vb.linked_clone          = true
        vb.customize ["modifyvm", :id, "--vram", "8"]
        vb.customize ["modifyvm", :id, "--audio", "none"]
      end

      $forwarded_ports.each do |guest, host|
        node.vm.network "forwarded_port", guest: guest, host: host, auto_correct: true
      end

      # Turn off the implicit default share
      config.vm.synced_folder ".", "/vagrant", disabled: true

      ip = "#{$subnet}.#{i+100}"
      node.vm.network :private_network, ip: ip

      # Disable swap
      node.vm.provision "shell", inline: "swapoff -a"

      # Disable firewalld on RHEL-based distros
      if ["rhel8","rockylinux8","rhel9","rockylinux9"].include? $os
        node.vm.provision "shell", inline: "systemctl stop firewalld; systemctl disable firewalld"
      end

      # Run Ansible once after all machines are up
      if i == $kafka_node_instances

        # Sweet trick to find vagrant forwarded nat ssh ports ;)
        node.vm.provision "ansible" do |ansible|
          ansible.playbook           = "tests/internal/pwd-local.yml"
          ansible.become             = false
          ansible.compatibility_mode = "2.0"
          ansible.inventory_path     = "tests/internal/inventory.ini"
          ansible.raw_arguments      = [
            "--connection=local",
            "--inventory=127.0.0.1",
            "--limit=127.0.0.1",
            "-i", "ansible_hosts"
          ]
          ansible.force_remote_user  = false
          ansible.host_key_checking  = false
        end

        node.vm.provision "ansible" do |ansible|
          ansible.playbook           = $playbook
          ansible.become             = true
          ansible.compatibility_mode = "2.0"
          ansible.config_file        = "tests/ansible.cfg"
          ansible.extra_vars         = $extra_vars
          ansible.inventory_path     = $inventory
          ansible.limit              = "all,localhost"
          ansible.raw_arguments      = [
            "--forks=#{$kafka_node_instances}",
            "--flush-cache",
            "-e", "ansible_become_pass=vagrant",
            "-e", "ansible_ssh_common_args=-F#{$vagrant_dir}/ssh-config"
          ]
          ansible.tags               = [$ansible_tags] if $ansible_tags != ""
          ansible.verbose            = $ansible_verbosity
          ansible.ask_become_pass    = false
          ansible.ask_vault_pass     = false
          ansible.force_remote_user  = false
          ansible.host_key_checking  = false
        end
      end
    end
  end
end
