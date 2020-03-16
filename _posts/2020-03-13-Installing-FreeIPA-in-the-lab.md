---
layout: post
title: "Installing FreeIPA, Pihole and Cockpit in the lab"
author: "Joakim Lönnegren"
categories: blog
tags: [Raspberry Pi, FreeIPA, Pihole, Cockpit]
image:
  feature: expanded-homelab/feature.jpg
  teaser: expanded-homelab/teaser.jpg
---

After expanding my machine park with a new Raspberry Pi 4 I started looking at installing some infrastructure for development at home as well as trying out some open source services. This includes using FreeIPA for logins and home directories over NFS as well as Pihole for DNS and ad blocking.

I created a couple of ansible roles for these which I gathered on [Github](https://github.com/joalon/home-lab)

## Hardware
* 1 x Raspberry Pi 4
* 1 x Raspberry Pi 3b
* Raspberry Pi Accessories (power supplies, network cables, fans/heat sinks, cases)
* Netlink switch
* Laptop

## SD-card configuration
Since I'm going to use FreeIPA as the domain controller I thought I'd use CentOS as the base operating system. They don't have an officially supported image but I burnt the ["RaspberryPI-Minimal-7-1908"](http://isoredirect.centos.org/altarch/7/isos/armhfp/CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-1908-sda.raw.xz) to an SD-card and it booted without any trouble. When I insert an SD-card in my computer it shows up as /dev/sdb:

```fish
wget http://isoredirect.centos.org/altarch/7/isos/armhfp/CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-1908-sda.raw.xz
xzcat ~/Downloads/CentOS-Userland-7-armv7hl-RaspberryPI-Minimal-4-1908-sda.raw.xz | sudo dd of=/dev/sdb bs=64k oflag=dsync status=progress
```

After booting it was time to set things up. [Here's](https://www.reddit.com/r/raspberry_pi/comments/dun1bk/centos7_raspberry_pi_4_install_w_docker/) a good post on Reddit which I mostly followed for setting up the system afterwards. Here's a quick recap:

```fish
ssh root@192.168.0.125   # Used default password 'centos'
passwd
rootfs-expand
```

Since I'm not going to use the Wifi or Bluetooth in the near future I decided to turn them off:

```fish
cat > /etc/modprobe.d/raspi-blklst.conf <<EOF

#wifi
blacklist brcmfmac
blacklist brcmutil

#bt
blacklist btbcm
blacklist hci_uart
EOF
```

Then I found how to [enable EPEL](https://wiki.centos.org/SpecialInterestGroup/AltArch/armhfp#How_Can_I_Enable_EPEL_7_on_armhfp_.3F) on the CentOS AltArch SIG:

```fish
cat > /etc/yum.repos.d/epel.repo <<EOF
[epel]
name=Epel rebuild for armhfp
baseurl=https://armv7.dev.centos.org/repodir/epel-pass-1/
enabled=1
gpgcheck=0
EOF
```

And then updating the system: `yum update -y`.

After the update I disabled chrony and enabled ntpd instead (Apparently FreeIPA prefers ntpd):

```fish
systemctl disable chronyd
systemctl stop chronyd
yum install -y ntp
```

I changed the pool servers in the ntp config file /etc/ntp.conf to [the south American ones](https://www.pool.ntp.org/zone/south-america).

```fish
timedatectl set-timezone "America/Sao_Paulo"
systemctl enable --now ntpd
ntpdate -u 0.south-america.pool.ntp.org
```

Setup FirewallD:

```fish
yum install -y firewalld
systemctl enable --now firewalld
```

And finally set the hostname for each host with `hostnamectl set-hostname` and reboot.

## FreeIPA
I wanted to have an Identity solution in the environment to get stuff like single sign on and TLS certificate handling for internal services and I settled for FreeIPA, the open source version of Red Hats IdM (Identity Management).

Installing the FreeIPA master took a couple of tries to get right, I kept running into a bug where the json API would time out during installation so it wouldn't install correctly. After some time searching I found [this issue](https://pagure.io/freeipa/issue/7337) which was similar where the author resolved it by adding a GSSAPI config option to the `/etc/httpd/conf.d/ipa.conf` file. So, while the `ipa-server-install` command ran, I waited for that file to be created and then I added `GssapiDelegCcacheEnvVar KRB5CCNAME` under the `Location "/ipa"`. I also increased the installation timeout, since this has [historically](https://pagure.io/freeipa/issue/6268) been a problem when installing FreeIPA on a Pi. After these modifications I ran the following commands on my `ipa.home.joalon.se` node:

```fish
yum install -y ipa-server ipa-server-dns bind-dyndb-ldap

for service in ntp http https ldap ldaps kerberos kpasswd dns; do firewall-cmd --permanent --add-service=$service; done
firewall-cmd --reload

ipa-server-install --setup-dns
```

To verify:

```fish
kinit admin
ipa user-find admin
```

![IPA master verification](/images/expanded-homelab/ipa-master-verification.png)

To test this I installed the FreeIPA client on my pihole node according to [these instructions](https://www.howtoforge.com/tutorial/how-to-install-freeipa-client-on-centos-7/).

On the IPA master I ran:

```fish
kinit admin
ipa dnsrecord-add home.joalon.se client --a-rec 192.168.0.126
```

And on the client:

```fish
cat > /etc/resolv.conf <<EOF
search home.joalon.se
nameserver 192.168.0.125
EOF

firewall-cmd --add-service=dns --permanent
firewall-cmd --reload

hostname-ctl set-hostname pihole.home.joalon.se

echo "192.168.0.125     ipa.home.joalon.se     ipa" >> /etc/hosts

reboot
```

After the reboot I installed the FreeIPA client:

```fish
yum -y install freeipa-client ipa-admintools
ipa-client-install --mkhomedir --force-ntpd --enable-dns-updates
```

![Client installation successful](/images/expanded-homelab/successful-client-enrollment.png)

Something to note with a new setup of FreeIPA is that the default [HBAC](https://www.freeipa.org/page/Howto/HBAC_and_allow_all) (Host Based Access Control) rule allows all users access to all machines. This can be tweaked by disabling the default rule and create some host groups to use with new rules. I'll leave the default for now.

## Cockpit
This is a tool to do system administration in a web interface. I'll install this mostly for the graphs in the overview:

![Cockpit overview](/images/expanded-homelab/cockpit-overview.png)

```fish
yum install -y realmd setroubleshoot
yum install -y cockpit cockpit-storaged cockpit-packagekit
```

Since this is a pretty sensitive service I wanted to get TLS working with a trusted cert from my new FreeIPA master. First I added a new private TLS key:

```fish
cd /etc/pki/tls/certs
make cockpit.key
mv cockpit.key ../private/cockpit.key
```

![TLS key generation](/images/expanded-homelab/tls-key-gen.png)

Then I could use this private key together with Certmonger, included in FreeIPA, to request a certificate for the cockpit service on that host:

```fish
ipa service-add cockpit/pihole.home.joalon.se
ipa-getcert request -K -P host/pihole.home.joalon.se -d /etc/cockpit/ws-certificates.d/ipa.crt
```

This failed for me on hosts with Selinux since the requested certificate doesn't get the 'cert_t' context when created. To get around this I added the selinux file context cert_t recursively to the directory:

```fish
semanage fcontext -a -t cert_t "/etc/cockpit/ws-certificates.d(/.*)?"
```

Finally, on my Arch laptop, I had to trust the FreeIPA cert by running:

```fish
scp root@ipa.home.joalon.se:~/ipa-ca.crt .
sudo trust anchor --store ipa-ca.crt
```

After this I got the green TLS checkbox when visiting the web interface:

![Trusted Cockpit interface](/images/expanded-homelab/trusted-cockpit.png)

## Pihole
This service was the easiest one to set up. After the common configuration I only had to run the install script and create the TLS configuration:

```fish
wget -O pihole-install.sh https://install.pi-hole.net
bash pihole-install.sh
```

To request a new TLS cert from FreeIPA for the web interface I used nearly the same commands as when configuring Cockpit. Except I also got the public CA trust chain from the FreeIPA master and converted it to a usable format:

```fish
pushd /etc/pki/tls/certs
make pihole-w-pass.key
openssl rsa -in pihole-w-pass.key -out pihole.home.joalon.se.key
mv pihole.home.joalon.se.key /etc/pki/tls/private/
popd

CERT_FILE=/etc/pki/tls/certs/pihole.home.joalon.se.pem
KEY_FILE=/etc/pki/tls/private/pihole.home.joalon.se.key

ipa service-add pihole/pihole.home.joalon.se
ipa-getcert request -f ${CERT_FILE} -k ${KEY_FILE} --principal pihole/$(hostname) -C "cat ${KEY_FILE} >> ${CERT_FILE}"

scp root@ipa:~/ca-agent.p12 /tmp/
openssl pkcs12 -in /tmp/ca-agent.p12 -out /etc/pki/tls/certs/ipa-ca-chain.pem 
systemctl restart lighttpd
```

After getting the combined key and certificate I followed the official [Pihole instructions](https://discourse.pi-hole.net/t/enabling-https-for-your-pi-hole-web-interface/) to configure lighttpd to use them. In short I added the following code snippet to `/etc/lighttpd/external.conf`:

```fish
$HTTP["host"] == "pihole.home.joalon.se" {
  # Ensure the Pi-hole Block Page knows that this is not a blocked domain
  setenv.add-environment = ("fqdn" => "true")

  $SERVER["socket"] == ":443" {
    ssl.engine = "enable"
    ssl.pemfile = "/etc/pki/tls/certs/pihole.home.joalon.se.pem"
    ssl.ca-file =  "/etc/pki/tls/certs/ipa-ca-chain.pem"
    ssl.honor-cipher-order = "enable"
    ssl.cipher-list = "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH"
    ssl.use-sslv2 = "disable"
    ssl.use-sslv3 = "disable"       
  }

  # Redirect HTTP to HTTPS
  $HTTP["scheme"] == "http" {
    $HTTP["host"] =~ ".*" {
      url.redirect = (".*" => "https://%0$0")
    }
  }
}
```

![Working pihole interface](/images/expanded-homelab/pihole-working-interface.png)

https://rcritten.wordpress.com/2018/11/26/how-do-i-get-a-certificate-for-my-web-site-with-ipa/

After verifying Pihole I added it as the global forwarder in FreeIPAs interface:

![Global forwarder](/images/expanded-homelab/global-forwarder.png)

## Molecule
[Molecule](https://molecule.readthedocs.io/) is a tool for speeding up the development cycle of ansible roles. When running molecule it starts a container, applies the role and then allows for running unit- or integration tests as well as logging in and poking around with a shell. It's still a bit clunky to get to work and it takes a minute or two to spin each run up but it's loads better than manually resetting a virtual machine. A typical run looks like this:

![Running Molecule](/images/expanded-homelab/running-molecule.png)

When developing the Ansible roles for these services, the biggest initial issue was getting Systemd to work in a docker container. I tried the suggestions from the [Molecule documentation](https://molecule.readthedocs.io/en/latest/examples.html), as well as from [Jeff Geerling's blog post](https://www.jeffgeerling.com/blog/2018/testing-your-ansible-roles-molecule) and a few other ideas until I found a [docker systemd image from lean-delivery](https://github.com/lean-delivery/docker-systemd).

Since I already have docker setup on my laptop,  I ran these commands to scaffold a new Ansible role:

```fish
pip3 install --user molecule
molecule init role joalon.freeipa-client
cd joalon.freeipa-client
```

And changed the `molecule/default/molecule.yml` to use the lean-delivery docker image:

```fish
- name: instance
    image: leandelivery/docker-systemd:centos7
    privileged: True
    security_opts:
      - seccomp=unconfined
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    tmpfs:
      - /tmp
      - /run
    capabilities:
      - SYS_ADMIN
```

Big disclaimer, using privileged docker containers and/or the SYS_ADMIN capability is like giving the image developer root access to the host system. I feel bad doing it but I didn't get it to work any other way.

After these steps I could incrementally develop a role:

```fish
# Do some changes to the ansible role
molecule converge

# Destroy the current container
molecule destroy

# Create it again and check that the role is idempotent
molecule converge
molecule converge
```

## Summary
The home lab took a bit longer to setup than I anticipated but I'm very glad with the automation and identity management. If I get around to enrolling my laptop into the domain I'll be able to use Ansible's dynamic inventory to get the hosts list from FreeIPA using [this script](https://github.com/ansible/ansible/blob/devel/contrib/inventory/freeipa.py). Anyway, thanks for reading!
