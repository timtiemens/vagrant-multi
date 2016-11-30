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
keys.each do |k|
  if ENV[k.upcase] != nil then
    puts "Overide from environment variable: " + k.upcase + " = " + ENV[k.upcase]
    if /^\d+/.match(ENV[k.upcase])
      CONFIG[k] = Integer(ENV[k.upcase])
    else
      CONFIG[k] = ENV[k.upcase]
    end
  end
end


# number of instances
num_instances = CONFIG['instances_ip'].length

CLUSTER_PREFIX = "cluster"

# TODO: for loop instances_ip, build up etc_hosts variable
etc_hosts = "# added by Vagrantfile\n"
(1..num_instances).each do |i|
    if i > 1
      nodename = "#{CLUSTER_PREFIX}-node%03d" % (i - 1)
    else
      nodename = "#{CLUSTER_PREFIX}-master"
    end  
  etc_hosts = "#{etc_hosts}\n#{CONFIG['instances_ip'][i - 1]}  #{nodename}"
end

# Vagrant writes this to /etc/hosts:
# 127.0.1.1 cluster1.vagrant cluster1

$script = <<SCRIPT
# Remove any "127.0.?.1 cluster1.vagrant" entries, because we want the IP-FQDN
#   to be one of the instance-ip[] items
sed -i "/127.0.0.1 #{CLUSTER_PREFIX}/d" /etc/hosts
sed -i "/127.0.1.1 #{CLUSTER_PREFIX}/d" /etc/hosts
# our contents
cat >> /etc/hosts <<EOF
#{etc_hosts}
EOF
SCRIPT

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # manage /etc/hosts by hostmanager plugin(https://github.com/smdahlen/vagrant-hostmanager)
  # use vagrant-cachier to cache packages at local(https://github.com/fgrehm/vagrant-cachier)
#  config.hostmanager.enabled = true

  # use vagrant-cachier to cache packages at local(https://github.com/fgrehm/vagrant-cachier)
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end

  # nodes definition
  (1..num_instances).each do |i|
    if i > 1
      nodename = "#{CLUSTER_PREFIX}-node%03d" % (i - 1)
    else
      nodename = "#{CLUSTER_PREFIX}-master"
    end
    config.vm.define "#{nodename}" do |node|

      node.vm.box = CONFIG['vm_box']
      node_hostname=nodename
      #  was node_fqdn="#{CLUSTER_PREFIX}#{i}.vagrant"
      node_fqdn="#{nodename}.vagrant"
      node_ip="#{CONFIG['instances_ip'][i - 1]}"

      node.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", CONFIG['memory_size']]
	vb.customize ['modifyvm', :id, '--cpus', CONFIG['number_cpus']]
      end

      node.vm.network :private_network, ip: node_ip
      node.vm.hostname = node_fqdn

      # three levels up is the bigtop "home" directory.
      # the current directory has puppet recipes which we need for provisioning.
#      bigtop.vm.synced_folder "./../apache-bigtop/", "/bigtop-home"

      # We also add the bigtop-home output/ dir, so that locally built rpms will be available.
#      puts "Adding output/ repo ? #{enable_local_repo}"

      # carry on w/ installation
#      node.vm.provision :shell do |shell|
#        shell.path = "./../apache-bigtop/bigtop-deploy/vm/utils/setup-env-" + distro + ".sh"
#        shell.args = ["#{enable_local_repo}"]
#      end
      
      node.vm.provision "shell", inline: $script

      # Add the ip to FQDN and hostname mapping in /etc/hosts
#      node.hostmanager.aliases = "#{bigtop_fqdn} #{bigtop_hostname}"


    end

  end

end
