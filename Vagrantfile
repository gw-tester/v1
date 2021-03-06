# -*- mode: ruby -*-
# vi: set ft=ruby :
##############################################################################
# Copyright (c) 2020
# All rights reserved. This program and the accompanying materials
# are made available under the terms of the Apache License, Version 2.0
# which accompanies this distribution, and is available at
# http://www.apache.org/licenses/LICENSE-2.0
##############################################################################

$no_proxy = ENV['NO_PROXY'] || ENV['no_proxy'] || "127.0.0.1,localhost"
# NOTE: This range is based on vagrant-libvirt network definition CIDR 192.168.121.0/24
(1..254).each do |i|
  $no_proxy += ",192.168.121.#{i}"
end
$no_proxy += ",10.0.2.15,10.10.17.4"

$debug = ENV['DEBUG'] || "false"
$deployment_type = ENV['DEPLOYMENT_TYPE'] || "docker"
$multi_cni = ENV['MULTI_CNI'] || "multus"
$pkg_mgr = ENV['PKG_MGR'] || "k8s"
$enable_skydive = ENV['ENABLE_SKYDIVE'] || "false"
$enable_portainer = ENV['ENABLE_PORTAINER'] || "false"

Vagrant.configure(2) do |config|
  config.vm.provider :libvirt
  config.vm.provider :virtualbox

  config.vm.box = "generic/ubuntu1804"
  config.vm.box_check_update = false
  config.vm.synced_folder './', '/vagrant'

  [:virtualbox, :libvirt].each do |provider|
  config.vm.provider provider do |p|
      p.cpus = 2
      p.memory = 6144
    end
  end

  config.vm.provider "virtualbox" do |v|
    v.gui = false
  end

  config.vm.provider :libvirt do |v|
    v.cpu_mode = 'host-passthrough'
    v.random_hostname = true
    v.management_network_address = "192.168.121.0/24"
  end

  if ENV['http_proxy'] != nil and ENV['https_proxy'] != nil
    if Vagrant.has_plugin?('vagrant-proxyconf')
      config.proxy.http     = ENV['http_proxy'] || ENV['HTTP_PROXY'] || ""
      config.proxy.https    = ENV['https_proxy'] || ENV['HTTPS_PROXY'] || ""
      config.proxy.no_proxy = $no_proxy
      config.proxy.enabled = { docker: false, git: false }
    end
  end
  # Install requirements
  config.vm.provision 'shell', privileged: false, inline: <<-SHELL
    source /etc/os-release || source /usr/lib/os-release
    case ${ID,,} in
        ubuntu|debian)
            sudo apt-get update
            sudo apt-get install -y -qq -o=Dpkg::Use-Pty=0 curl
        ;;
    esac
    # NOTE: Shorten link -> https://github.com/electrocucaracha/pkg-mgr_scripts
    curl -fsSL http://bit.ly/install_pkg | PKG="docker make" bash
    if cat /proc/sys/net/ipv4/ip_forward | grep 0; then
        sudo sysctl -w net.ipv4.ip_forward=1
        sudo sed -i "s/#net.ipv4.ip_forward=.*/net.ipv4.ip_forward=1/g" /etc/sysctl.conf
    fi
  SHELL
  # Deploy services
  config.vm.provision 'shell', privileged: false do |sh|
    sh.env = {
      'DEBUG': "#{$debug}",
      'DEPLOYMENT_TYPE': "#{$deployment_type}",
      'MULTI_CNI': "#{$multi_cni}",
      'PKG_MGR': "#{$pkg_mgr}",
      'ENABLE_SKYDIVE': "#{$enable_skydive}",
      'ENABLE_PORTAINER': "#{$enable_portainer}",
      'HOST_IP': "10.10.17.4"
    }
    sh.inline = <<-SHELL
      set -o pipefail
      set -o errexit

      echo "export DEPLOYMENT_TYPE=$DEPLOYMENT_TYPE" | sudo tee --append /etc/environment
      echo "export MULTI_CNI=$MULTI_CNI" | sudo tee --append /etc/environment
      echo "export PKG_MGR=$PKG_MGR" | sudo tee --append /etc/environment

      cd /vagrant
      ./install.sh | tee ~/install.log
      ./deploy.sh | tee ~/deploy.log
    SHELL
  end
  config.vm.network :forwarded_port, guest: 8082, host: 8082
  config.vm.network :forwarded_port, guest: 9000, host: 9000
end
