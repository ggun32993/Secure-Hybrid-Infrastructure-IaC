# -*- mode: ruby -*-
# vi: set ft=ruby :

# จำลองระบบแล็บ (Gateway, Web, DB)
Vagrant.configure("2") do |config|
  # ใช้ Ubuntu 22.04 ทุกเครื่อง
  config.vm.box = "ubuntu/jammy64"

  # 1. เครื่อง Gateway (VPN และ Routing)
  config.vm.define "gateway" do |gateway|
    gateway.vm.hostname = "gateway"
    
    # ไอพีสำหรับต่อ VPN จากเครื่อง Windows
    gateway.vm.network "private_network", ip: "192.168.56.10"
    # วงแลนภายในเชื่อมไปเครื่อง Web และ DB
    gateway.vm.network "private_network", ip: "10.0.0.1", virtualbox__intnet: "secure_lan"

    # กำหนดสเปคเครื่องจำลอง
    gateway.vm.provider "virtualbox" do |vb|
      vb.name = "se-lab-gateway"
      vb.memory = "1024"
      vb.cpus = 1
    end

    # รันคอนฟิกด้วย Ansible
    gateway.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/playbook-gateway.yml"
      ansible.install = true
    end
  end

  # 2. เครื่อง Web Server (ในวงแลนภายใน)
  config.vm.define "web" do |web|
    web.vm.hostname = "web"
    
    # ต่อเข้าวงแลนภายใน
    web.vm.network "private_network", ip: "10.0.0.11", virtualbox__intnet: "secure_lan"

    web.vm.provider "virtualbox" do |vb|
      vb.name = "se-lab-web"
      vb.memory = "1024"
      vb.cpus = 1
    end

    # รันคอนฟิกเครื่องเว็บ
    web.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/playbook-web.yml"
      ansible.install = true
    end
  end

  # 3. เครื่อง Database (ในวงแลนภายใน)
  config.vm.define "db" do |db|
    db.vm.hostname = "db"
    
    # ต่อเข้าวงแลนภายใน
    db.vm.network "private_network", ip: "10.0.0.12", virtualbox__intnet: "secure_lan"

    db.vm.provider "virtualbox" do |vb|
      vb.name = "se-lab-db"
      vb.memory = "1024"
      vb.cpus = 1
    end

    # รันคอนฟิกฐานข้อมูล
    db.vm.provision "ansible_local" do |ansible|
      ansible.playbook = "ansible/playbook-db.yml"
      ansible.install = true
    end
  end
end
