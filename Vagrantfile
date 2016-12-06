# -*- mode: ruby -*-
# vi: set ft=ruby :

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

require "yaml"

_config = YAML.load(File.open(File.join(File.dirname(__FILE__), "vagrantconfig.yaml"), File::RDONLY).read)
CONFIG = _config

# Override vagrant configurations using environment variables
keys = CONFIG.keys
keys.each do |outerk|
  CONFIG[outerk].each do |inneritem|
    innerk = inneritem[0]
    key = "#{outerk}_#{innerk}"
    val = ENV[key]
    if val == nil then
      key = key.upcase
      val = ENV[key]
    end
    if val != nil then
      puts "Overide from environment variable: " + key + " = " + val
      if /^\d+/.match(val)
        val = Integer(val)
      end
      CONFIG[outerk][innerk] = val
    end
  end
end

# i = 1 .. N      index = 0 .. N-1
def getConfigValue(index, key)
  config = CONFIG
  retval = "ERROR for #{index} for #{key}"
  if config.has_key?('defaults')
    defaultReturn = config['defaults'][key]
    retval = defaultReturn
  else
    puts "ERROR - missing 'defaults' section in configuration"
  end
  if config.has_key?('instances')
    item = config['instances'][index]
    if item.has_key?(key)
      retval = item[key]
    end
  end

#  puts "Return for instances[#{index}][#{key}] is #{retval}"
  return retval
end

CLUSTER_PREFIX = "cluster"

# i = 1 .. N      index = 0 .. N-1
def getHostName(i)
  # if you have more than 99 nodes, then this is not the tool for you:
  if i > 1
    nodename = "#{CLUSTER_PREFIX}-node%02d" % (i - 1)
  else
    nodename = "#{CLUSTER_PREFIX}-master"
  end
#  puts "Return for getHostName(#{i}) is #{nodename}"
  return nodename
end

# i = 1 .. N      index = 0 .. N-1
def getIpAddress(i)
  return CONFIG['instances'][i - 1]['ip_address']
end

# i = 1 .. N      index = 0 .. N-1
def getSyncFolders(index)
  retval = getConfigValue(index, 'synced_folders')
  if retval.nil? || retval =~ /ERROR/
    retval = []
  end
  return retval
end

# i = 1 .. N      index = 0 .. N-1
def getExtraDisks(index)
  retval = getConfigValue(index, 'extra_disks')
  if retval.nil? || retval =~ /ERROR/
    retval = []
  end
  return retval
end

def getFileToDisk(disk)
  filename = disk['file_name']
  if filename
    # Note: hard-coded .vmdk
    if ! filename.end_with? ".vmdk"
      filename = "#{filename}.vmdk"
    end
  
    # TODO: absolute path? for now just leave it implied as "."
    file_to_disk = filename
  else
    file_to_disk = nil
  end
  
  return file_to_disk
  
end

# number of instances
num_instances = CONFIG['instances'].length


#
# build up the /etc/hosts content:
# Note: Vagrant writes this to /etc/hosts:
#        127.0.1.1 cluster1.vagrant cluster1
# Note: Remove any "127.0.?.1 cluster1.vagrant" entries, because we want the IP-FQDN
#   to be one of the instances[:ip_address:] items, you could use sed:
#     sed -i "/127.0.0.1 #{CLUSTER_PREFIX}/d" /etc/hosts
#     sed -i "/127.0.1.1 #{CLUSTER_PREFIX}/d" /etc/hosts
# But currently, this is done because the entire /etc/hosts file is replaced:
#
etc_hosts_fixed = <<SCRIPT
# fixed, by Vagrantfile
127.0.0.1 localhost

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

# dynamic, by Vagrantfile
SCRIPT

etc_hosts_dynamic = ""
(1..num_instances).each do |i|
   nodename = getHostName(i)   
   ipaddress = getIpAddress(i)
  etc_hosts_dynamic = "#{etc_hosts_dynamic}\n#{ipaddress}  #{nodename}"
end

$script = <<SCRIPT
# our contents
cat > /etc/hosts <<EOF
#{etc_hosts_fixed}
#{etc_hosts_dynamic}
EOF
SCRIPT




VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  #
  # Vagrant is totally messed up -
  #  there is no accepted way to reliably "create the .vmdk if it does not exist" when
  #    you are inside Vagrant.configure...
  #  so - look at the "puts", and how many times it prints on a single "vagrant up"...
  #
  # NOTE: *** on 'vagrant destroy', extra disks are deleted ***
  
  # nodes definition
  (1..num_instances).each do |i|
#    puts "defineloop, i is #{i}"
    index = i - 1    
    nodename = getHostName(i)

    
    config.vm.define "#{nodename}" do |node|
      memory_size = getConfigValue(index, 'memory_size')
      number_cpus = getConfigValue(index, 'number_cpus')
      node_ip     = getIpAddress(i)
      
      node.vm.box = getConfigValue(index, 'vm_box')
      
      node_hostname = nodename
      node_fqdn = "#{nodename}.vagrant"

      node.vm.hostname = node_fqdn

      node.vm.provider :virtualbox do |vb|
        # memory and CPU:
        vb.customize ["modifyvm", :id, "--memory", memory_size]
	vb.customize ['modifyvm', :id, '--cpus', number_cpus]

        # extra disks:
        disks = getExtraDisks(index)
        disks.each do |disk|
          # How many times do you think this prints "i is 3"??
          #   If you said "two or three or four", then you are correct.
          #   If you said "two" or "three" or "four", then you get partial credit.
          #  Vagrant stinks.
          # puts " +++ i is #{i}, disk for #{index} name #{disk['name']}"
          file_to_disk = getFileToDisk(disk)
          size_in_mb = disk['size_in_mb']
          # NOTE: conversion here "4096" gave 4293MB /dev/sdb, 4.0G ext2 file system
          size_for_createhd = size_in_mb
          
          if (ARGV[0] == "up") && ! File.exist?(file_to_disk)
            puts "+++ Creating disk #{file_to_disk} size #{size_in_mb}"
            vb.customize ["createhd",
                          "--filename", file_to_disk,
                          "--size", size_for_createhd,
                          "--format", "VMDK"]
          end

          # TODO: on "vagrant destroy", make the disk file stay around?
          vb.customize ['storageattach', :id,
                        '--storagectl', 'SATAController',
                        '--port', 1,
                        '--device', 0,
                        '--type', 'hdd',
                        '--medium', file_to_disk]          
        end
      end

      node.vm.network :private_network, ip: node_ip

      
      # sync folders:
      folders = getSyncFolders(index)
      folders.each do |folder|
        node.vm.synced_folder folder['src'], folder['dest'], folder['options']
      end

      # /etc/hosts:
      node.vm.provision "shell", inline: $script


    end

  end

end
