$script = <<-SCRIPT

# Install Ansible inside corcho
apt-get install -y python3-pip sshpass
su - vagrant -c 'pip3 install --user ansible jmespath'

# Allow connections from ansible to corcho
sed -i 's/PasswordAuthentication\ no/PasswordAuthentication\ yes/g' /etc/ssh/sshd_config
systemctl restart sshd

# Install QDPaaS
su - vagrant -c 'ansible-playbook -i /vagrant/hosts /vagrant/install.yml --tags install --limit corcho'

# Create VPN config for clients
su - vagrant -c 'ansible-playbook -i /vagrant/hosts /vagrant/install.yml --tags build_ovpn --limit corcho'


#echo "Nextcloud installation and external storage config"
cat <<EOF >>/tmp/external_storage.json
[
    {
        "mount_id": 1,
        "mount_point": "\/docker-share",
        "storage": "\\\\OC\\\\Files\\\\Storage\\\\Local",
        "authentication_type": "null::null",
        "configuration": {
            "datadir": "\/docker-share"
        },
        "options": {
            "encrypt": true,
            "previews": true,
            "enable_sharing": false,
            "filesystem_check_changes": 1,
            "encoding_compatibility": false,
            "readonly": false
        },
        "applicable_users": [],
        "applicable_groups": []
    }
]
EOF

# Execute right after firt boot in order to finish nextcloud installation
#docker exec -u 33 nextcloud_app_1 php occ maintenance:install \
#--database "mysql" --database-host "db" --database-name "nextcloud" \
#--database-user "nextcloud" --database-pass "1234" \
#--admin-user "admin" --admin-pass "1234"
#sleep 30
#docker exec -u 33 nextcloud_app_1 php occ app:enable files_external
#docker cp /tmp/external_storage.json nextcloud_app_1:/tmp/
#docker exec -u 33 nextcloud_app_1 php occ files_external:import /tmp/external_storage.json

# You'll also need to configure Emby to use /docker-share

SCRIPT


Vagrant.configure("2") do |config|
  config.vm.define "corcho" do |config|
    config.vm.box = "debian/buster64"
    config.vm.hostname = "corcho"

    # lan interface connected to clients
    config.vm.network "private_network", ip: "10.0.42.1"
    # access to the vpn server
    config.vm.network "forwarded_port", guest: 3342, host: 3342, protocol: "udp"
    # open this so transmission can see its port as open
    config.vm.network "forwarded_port", guest: 51413, host: 51413, protocol: "tcp"
    config.vm.network "forwarded_port", guest: 51413, host: 51413, protocol: "udp"

    config.vm.provider "virtualbox" do |vb|
      vb.memory = "2048"
    end

    config.vm.provision "shell", inline: $script
  end

end
