require "yaml"
vagrant_root = File.dirname(File.expand_path(__FILE__))
settings = YAML.load_file "#{vagrant_root}/settings.yaml"

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
IP_NW = IP_SECTIONS.captures[0]  # First 3 octets including the trailing dot
IP_START = Integer(IP_SECTIONS.captures[1])  # Last octet
NUM_CONTROL_NODES = settings["nodes"]["control"]["count"]
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.box_check_update = true

  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
    # Check if the controlplane entry already exists
    if ! (grep -q "controlplane" /etc/hosts && grep -q "node0" /etc/hosts); then
        # Update package list
        apt-get update -y

        # Add the controlplane entry
        echo "$IP_NW$((IP_START)) controlplane" >> /etc/hosts

        # Add worker nodes entries
        for i in `seq 1 ${NUM_WORKER_NODES}`; do
            echo "$IP_NW$((IP_START+i)) node0${i}" >> /etc/hosts
        done
    else
        echo "Controlplane entry already exists in /etc/hosts. Skipping update."
    fi
  SHELL

  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end

  config.vm.define "controlplane" do |controlplane|
    controlplane.vm.hostname = "controlplane"
    controlplane.vm.network "private_network", ip: settings["network"]["control_ip"]
    controlplane.vm.provider "libvirt" do |lv|
      lv.cpus = settings["nodes"]["control"]["cpu"]
      lv.memory = settings["nodes"]["control"]["memory"]
    end
    controlplane.vm.provision "shell",
      env: {
        "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
        "ENVIRONMENT" => settings["environment"],
        "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
        "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
        "OS" => settings["software"]["os"]
      },
      path: "scripts/common.sh"
    controlplane.vm.provision "shell",
      env: {
        "CALICO_VERSION" => settings["software"]["calico"],
        "CONTROL_IP" => settings["network"]["control_ip"],
        "POD_CIDR" => settings["network"]["pod_cidr"],
        "SERVICE_CIDR" => settings["network"]["service_cidr"]
      },
      path: "scripts/controlplane.sh"
  end

  (1..NUM_WORKER_NODES).each do |i|
    config.vm.define "node0#{i}" do |node|
      node.vm.hostname = "node0#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      node.vm.provider "libvirt" do |lv|
        lv.cpus = settings["nodes"]["workers"]["cpu"]
        lv.memory = settings["nodes"]["workers"]["memory"]
      end

      node.trigger.before :up do |trigger|
        trigger.ruby do
          until File.exist?(".vagrant/machines/controlplane/libvirt/private_key")
            puts "node0#{i}: Waiting for controlplane private key to be available."
            sleep 180
          end
          puts "node0#{i}: Private key is now available."
          FileUtils.cp(".vagrant/machines/controlplane/libvirt/private_key", "node0#{i}_private_key")
        end
      end

      node.trigger.after :up do |trigger|
        trigger.ruby do
          if File.exist?("node0#{i}_private_key")
            File.delete("node0#{i}_private_key")
          end
        end
      end

      node.vm.provision "shell", inline: <<-SHELL
        # Copy the private key and set permissions
        cp /vagrant/node0#{i}_private_key /home/vagrant/.ssh/id_rsa
        chown vagrant:vagrant /home/vagrant/.ssh/id_rsa
        chmod 600 /home/vagrant/.ssh/id_rsa

        # Generate the public key from the private key and add it to authorized_keys
        sudo -u vagrant ssh-keygen -y -f /home/vagrant/.ssh/id_rsa >> /home/vagrant/.ssh/authorized_keys

        # Optionally, remove the private key after use for security
        rm -f /vagrant/node0#{i}_private_key

        # Ensure the known_hosts file is present for the vagrant user
        sudo -u vagrant touch /home/vagrant/.ssh/known_hosts
        
        # Add controlplane host key to known_hosts
        sudo -u vagrant ssh-keyscan -H controlplane >> /home/vagrant/.ssh/known_hosts

        # Check for join.sh on controlplane
        echo "Waiting for join.sh to be available on controlplane..."
        controlplane_ip="controlplane" # Replace with actual IP or hostname if needed
        while ! ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /home/vagrant/.ssh/id_rsa vagrant@${controlplane_ip} '[ -f /vagrant/configs/join.sh ]'; do
          echo "join.sh not found. Checking again in 30 seconds..."
          sleep 30 # Adjust the sleep interval as needed
        done

        echo "join.sh found. Copying /vagrant/configs from controlplane to node0#{i}"
        sudo -u vagrant scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i /home/vagrant/.ssh/id_rsa -r vagrant@${controlplane_ip}:/vagrant/configs /vagrant/
      SHELL

      node.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
          "OS" => settings["software"]["os"]
        },
        path: "scripts/common.sh"
      node.vm.provision "shell", path: "scripts/node.sh"

      # Only install the dashboard after provisioning the last worker (and when enabled).
      if i == NUM_WORKER_NODES and settings["software"]["dashboard"] and settings["software"]["dashboard"] != ""
        node.vm.provision "shell", path: "scripts/dashboard.sh"
      end
    end
  end
end
