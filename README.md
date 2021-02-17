# QDPaaS

Quick & Dirty Platform as a Service: your own cloud and content delivery network.


# Requirements

QDPaaS can be run on your current Debian based system, but it can also run inside its own VM on Windows/Mac computers.

## Virtual machine

### Preparation

You need these apps installed on your computer:
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Vagrant](https://www.vagrantup.com/downloads.html)

### Install

In order to setup the VPN gateway, private cloud and streaming service just run:

```shell
vagrant up
```

## Native

The playbooks have been tested on Debian Buster and Ubuntu Desktop 20.04 and later.

### Preparation

1. Add your system to the _host_ file or use your own Ansible inventory.
2. Configure your network in a new file under *host_vars*. Its name must be the same as the hostname you're going to install QDPaaS to. You can find which variables are available looking at the _defaults_ folder on each role.

### Install

You'll need to add your computer to the _hosts_ file so Ansible can connect to it.

In order to setup the VPN gateway, private cloud and streaming service just run:

```shell
ansible-playbook -i hosts install.yml --tags install --limit your_computer
```


# Usage

Everything is published inside the VPN so you must get the client configuration file and import it into some openvpn client.

## Get the configuration file

In order to show the VPN config for _htpc_ client you can call ansible from outside the VM:

### Linux/Mac

Just run:

```shell
$ vagrant ssh corcho -- '.local/bin/ansible-playbook -i /vagrant/hosts /vagrant/install-vagrant.yml --tags build_ovpn --limit corcho -e "vpn_clients=htpc"' | sed 's/\\n/\n/g'
```

### Windows

If you're on windows the output will be messed up, so you have to run two separate commands:

```shell
C:\Users\luser\Downloads\QDPaaS-0.1.0>vagrant ssh corcho -- '.local/bin/ansible-playbook -i /vagrant/hosts /vagrant/install.yml --tags build_ovpn --limit corcho -e "vpn_clients=htpc"'
C:\Users\luser\Downloads\QDPaaS-0.1.0>vagrant ssh corcho -- 'cat /tmp/htpc.ovpn'
```

And get everything between this:

```shell
TASK [easy-rsa : debug] ********************************************************
ok: [corcho] => {
    "msg": "client
dev tun
remote 10.0.2.15 3342 udp
```

And this:

```shell
POCWWAyQ5DZXa0AU+5bKa2rJzf/ewa22Ja3CEV8JoZJsgoVsf5tMOeiZLGMr5P1G
WBF76w==
-----END CERTIFICATE-----
</ca>"
}
```

## Add a device to the VPN

You have to copy&paste the info into a file named _htpc.ovpn_ so after removing the unnecesary bits it looks like this:

```shell
client
dev tun
remote 10.0.2.15 3342 udp
...
POCWWAyQ5DZXa0AU+5bKa2rJzf/ewa22Ja3CEV8JoZJsgoVsf5tMOeiZLGMr5P1G
WBF76w==
-----END CERTIFICATE-----
</ca>
```

You can confirm it has the correct certificate because it has the asked client name on it:

```shell
        Subject: CN=htpc
```

With that config file you can connect to the virtual machine you've just created. There are OpenVPN clients available for all platforms.


# Features

There's a couple of services running inside the vpn:
- [Transmission](http://transmission.qdpaas.vpn): donwload content from internet
- [Nextcloud](http://nube.qdpaas.vpn): organize and share your content
- [Emby](http://emby.qdpaas.vpn): view your content on any device
- [Grafana](http://grafana.qdpaas.vpn): show stats and get alerts

Default user/password is admin/admin.


# Backup of the VPN configuration

In order to move QDPaaS from one machine to another, it is necessary to do a backup of the VPN configuration, otherwise the clients will not be able to connect to the new installation, since it will use new certificates.

## Backup

```shell
vagrant@corcho:/etc/openvpn$ sudo tar zcvf /tmp/vpn-server.tar.gz QDPaaS.conf server/
QDPaaS.conf
server/
server/ca.crt
server/client-config/
server/QDPaaS.key
server/crl.pem
server/QDPaaS.crt
server/ta.key
server/dh2048.pem
vagrant@corcho:/etc/openvpn$ 
```

Download the tar.gz file.

## Restore

Put the tar.gz file inside _backup/vpn-server/_ and then execute:

```shell
vagrant@corcho:~$ ansible-playbook -i /vagrant/hosts /vagrant/install.yml --tags install --limit corcho -e restore_backup=true
```


# Backup docker volumes

Remember to check there's enough free space on the host running Ansible before making a backup:

```shell
ansible-playbook -i hosts backup-restore.yml --limit corcho
```

When restoring a backup, today is used as default date:

```shell
ansible-playbook -i hosts backup-restore.yml --limit corcho -e docker_date=20210214
```

## Manual backup

For making a backup of *emby_config* put inside a file _Dockerfile_ on QDPaaS machine:

```shell
FROM alpine
ENTRYPOINT tar -zcf /backup/emby_config-`date +%Y-%m-%d`.tgz -C /volume ./
```

Build the container which performs the backup and restore procedures:

```shell
docker build -t QDPaaS-backup .
```

Run the backup of *emby_config* volume:

```shell
docker run --rm \
-v emby_config:/volume:ro \
-v /tmp:/backup \
backup_qdpaas '/bin/tar -zcf /backup/emby_config-2021-02-14.tgz -C /volume ./'
```

/tmp on QDPaaS must have enough space to save the backup. 

In order to restore, first upload the _tgz_ file to the /tmp of the QDPaaS machine and run:

```shell
docker run --rm \
-v emby_config:/volume \
-v /tmp:/backup \
backup_qdpaas 'rm -rf /volume/* /volume/..?* /volume/.[!.]* ; tar -C /volume/ -zxf /backup/emby_config-2021-02-14.tgz'
```
