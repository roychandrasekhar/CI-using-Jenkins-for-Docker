Vagrant.configure("2") do |config|
  # Master node
  config.vm.define "jenkins-machine" do |master|
    master.vm.box = "ubuntu/focal64"
    master.vm.network "private_network", type: "static", ip: "192.168.50.40"
    master.ssh.insert_key = false
    master.vm.provider "virtualbox" do |v|
      v.memory = 2048
    end
    master.vm.provision "shell", inline: <<-SHELL
      sudo apt-get update
      sudo apt-get install -y python unzip net-tools
      sudo echo jenkins-machine > /etc/hostname
      sudo hostnamectl set-hostname jenkins-machine

      # Install Packer
      wget https://releases.hashicorp.com/packer/1.7.4/packer_1.7.4_linux_amd64.zip
      unzip packer_1.7.4_linux_amd64.zip
      sudo mv packer /usr/local/bin/
      rm packer_1.7.4_linux_amd64.zip

      # Install daemon (required dependency for Jenkins)
      sudo apt-get install openjdk-11-jre -y
      curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
      sudo echo "deb [trusted=yes] https://pkg.jenkins.io/debian binary/" > /etc/apt/sources.list.d/jenkins.list
      sudo apt-get update
      sudo apt-get install jenkins -y

      sudo systemctl enable --now jenkins
      sudo apt-get install git
	  
	  # Install Maven
      # sudo apt-get install maven -y
	  
	  # Install Docker
      sudo apt-get install -y docker.io
      sudo usermod -aG docker $USER

      # Open firewall for Jenkins (port 8080)
      sudo ufw allow 8080
      sudo ufw enable
	  
	  sudo mkdir -p /etc/systemd/resolved.conf.d/
	  sudo echo "[Resolve]" > /etc/systemd/resolved.conf.d/custom.conf
	  sudo echo "DNS=8.8.8.8" >> /etc/systemd/resolved.conf.d/custom.conf
	  sudo echo "DNS=8.8.4.4" >> /etc/systemd/resolved.conf.d/custom.conf
	  sudo systemctl restart systemd-resolved
	  
    SHELL
  end
end
