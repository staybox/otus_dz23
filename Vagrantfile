# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
 :web => {
        :box_name => "centos/7"
  }

}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        config.vm.provider "virtualbox" do |v|
          v.memory = 256
        end

        #boxconfig[:net].each do |ipconf|
        #  box.vm.network "private_network", ipconf
        #end
        
        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
                cp ~vagrant/.ssh/auth* ~root/.ssh
        SHELL
        
        # Директивы говорящие что надо использовать вход в гостевые машины используя логин и пароль
        #config.ssh.username = 'vagrant'
        #config.ssh.password = 'vagrant'
        #config.ssh.insert_key = false
        #config.ssh.connect_timeout = 5

        case boxname.to_s
        when "web"
          box.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            
            # Установка софта
            sudo yum install -y epel-release; sudo yum install -y tcpdump wget elinks nginx nano iptables-services; sudo systemctl enable iptables && sudo systemctl start iptables;

            sudo cp /vagrant/config/default.conf /etc/nginx/conf.d/; sudo cp /vagrant/config/nginx.conf /etc/nginx/

            # Отключение файервола
            sudo setenforce 0
            sudo sed -i 's/=enforcing/=disabled/g' /etc/selinux/config

            # Очищаем таблицы iptables
            sudo iptables -P INPUT ACCEPT
            sudo iptables -P FORWARD ACCEPT
            sudo iptables -P OUTPUT ACCEPT
            sudo iptables -t nat -F
            sudo iptables -t mangle -F
            sudo iptables -F
            sudo iptables -X
            sudo service iptables save

            systemctl enable nginx && systemctl start nginx

            SHELL
        
        end

      end

  end
  
end