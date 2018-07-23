# -*- mode: ruby -*-
# vi: set ft=ruby :

$script = <<SCRIPT
# Update apt and get dependencies
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y unzip curl vim \
    apt-transport-https \
    ca-certificates \
    software-properties-common
# Download Nomad
NOMAD_VERSION=0.8.1
echo "Fetching Nomad..."
cd /tmp/
curl -sSL https://releases.hashicorp.com/nomad/${NOMAD_VERSION}/nomad_${NOMAD_VERSION}_linux_amd64.zip -o nomad.zip
echo "Fetching Consul..."
CONSUL_VERSION=1.0.7
curl -sSL https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip > consul.zip
echo "Installing Nomad..."
unzip nomad.zip
sudo install nomad /usr/bin/nomad
sudo mkdir -p /etc/nomad.d
sudo chmod a+w /etc/nomad.d
# Set hostname's IP to made advertisement Just Work
#sudo sed -i -e "s/.*nomad.*/$(ip route get 1 | awk '{print $NF;exit}') nomad/" /etc/hosts
echo "Installing Docker..."
if [[ -f /etc/apt/sources.list.d/docker.list ]]; then
    echo "Docker repository already installed; Skipping"
else
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
fi
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y docker-ce
# Restart docker to make sure we get the latest version of the daemon if there is an upgrade
sudo service docker restart
# Make sure we can actually use docker as the vagrant user
sudo usermod -aG docker vagrant
echo "Installing Consul..."
unzip /tmp/consul.zip
sudo install consul /usr/bin/consul
(
cat <<-EOF
	[Unit]
	Description=consul agent
	Requires=network-online.target
	After=network-online.target
	
	[Service]
	Restart=on-failure
	ExecStart=/usr/bin/consul agent -dev
	ExecReload=/bin/kill -HUP $MAINPID
	
	[Install]
	WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/consul.service
sudo systemctl enable consul.service
sudo systemctl start consul
for bin in cfssl cfssl-certinfo cfssljson
do
	echo "Installing $bin..."
	curl -sSL https://pkg.cfssl.org/R1.2/${bin}_linux-amd64 > /tmp/${bin}
	sudo install /tmp/${bin} /usr/local/bin/${bin}
done
echo "Installing autocomplete..."
nomad -autocomplete-install
SCRIPT

Vagrant.configure(2) do |config|
  # Set box type for all machines
  config.vm.box = "bento/ubuntu-16.04" # 16.04 LTS

  # Provision all machines with script
  config.vm.provision "shell", inline: $script, privileged: false

  # Install docker
  config.vm.provision "docker"

  # Server 1
  config.vm.define "s1" do |s1|
      s1.vm.hostname = "s1"
      config.vm.provision "file", source: "./nomad/server1.hcl", destination: "$HOME/server1.hcl"
      s1.vm.network "private_network", ip: "172.20.20.10"
      # Expose Nomad UI port to localhost
      config.vm.network "forwarded_port", guest: 4646, host: 4646, auto_correct: true
  end

  # Client 1
  config.vm.define "c1" do |c1|
      c1.vm.hostname = "c1"
      config.vm.provision "file", source: "./nomad/client1.hcl", destination: "$HOME/client1.hcl"
      c1.vm.network "private_network", ip: "172.20.20.11"
  end

  # Client 2
  config.vm.define "c2" do |c2|
      c2.vm.hostname = "c2"
      config.vm.provision "file", source: "./nomad/client2.hcl", destination: "$HOME/client2.hcl"
      c2.vm.network "private_network", ip: "172.20.20.12"
  end
end