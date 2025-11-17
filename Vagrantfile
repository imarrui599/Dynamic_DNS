# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # Usaremos una caja base de Ubuntu Server
  config.vm.box = "ubuntu/jammy64"
  
  # --- Servidor DNS (IP estática) ---
  config.vm.define "dns" do |dns|
    dns.vm.hostname = "dns.example.test"
    dns.vm.network "private_network", ip: "192.168.58.10"
  end

  # --- Servidor DHCP (IP estática) ---
  config.vm.define "dhcp" do |dhcp|
    dhcp.vm.hostname = "dhcp.example.test"
    dhcp.vm.network "private_network", ip: "192.168.58.20"
  end

  # --- Cliente (Obtendrá IP dinámica) ---
  # Se coloca en la red privada, pero la IP la asignará el servidor DHCP.
  config.vm.define "cl" do |cl|
    cl.vm.hostname = "cliente-test"
    cl.vm.network "private_network", auto_config: false 
  end

  # Configuraciones de rendimiento
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "1024"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end
end