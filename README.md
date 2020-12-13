# Proxmox-RancherOS-Portainer-Ultimate-Setup

## Install Proxmox and a RancherOS VM with Portainer to manage docker containers

#### WARNING MAY BE INCORRECT AND INCOMPLETE, USE AT YOUR OWN RISK

I have tried this setup and although it works and may be good for ceratin circumstances I would advise using ubuntu as the base with docker, docker-compose, and portainer.
https://gist.github.com/mow4cash/626275e095f7f90898944a85d66b3be6



Link to my docker run file https://gist.github.com/mow4cash/6a25343cdeb0cd115f263dea0a3b623d


## Setup Proxmox

1. Install Proxmox 6.X iso
2. Console/SSH into Proxmox
3. nano /etc/apt/sources.list
4. edit the file to look like this
```
deb http://ftp.debian.org/debian buster main contrib
deb http://ftp.debian.org/debian buster-updates main contrib

# PVE pve-no-subscription repository provided by proxmox.com,
# NOT recommended for production use
deb http://download.proxmox.com/debian/pve buster pve-no-subscription

# security updates
deb http://security.debian.org buster/updates main contrib
```
5. apt update && apt dist-upgrade -y
6. reboot system

## Install RancherOS

1. Upload the RancherOS iso to (local)pve
2. Setup a VM with RancherOS ISO as CD. Give it at least 3gb ram to start. Rancher Server failed with low ram
3. Boot
4. From Console change password
-sudo bash
-passwd rancher
5. SSH to rancher@<host>
6. prepare your ssh keys with putty gen  
-vi cloud-config.yml   
7. paste the cloud config edited with your settings, make sure the pasted data is pated correctly, add your key in a single line and make sure the file has #cloud-config in the beginning
8. press exit exit :wq to save  
```
#cloud-config

rancher: rancheros
  network:
    interfaces:
      eth0:
        address: 10.68.69.92/24
        gateway: 10.68.69.1
        mtu: 1500
        dhcp: false
    dns:
      nameservers:
      - 1.1.1.1
      - 8.8.4.4

ssh_authorized_keys:
  - ssh-rsa <YOUR KEY>  
```
9. sudo ros config validate -i cloud-config.yml 
10. sudo ros install -c cloud-config.yml -d /dev/sda
11. Remove CD Image from VM, and then reboot.
12. SSH back into RancherOS (rancher@<IP>) using your new ssh private key  

## Create NFS Shares on FreeNAS

1. create a unix dataset called appsNFS with root and wheel as the user, set a quota for 50gb
2. create a nfs share to the dataset you created, select all dirs, mapall user:group to root:wheel
3. enable nfs sharing and select nfsv4, allow non-root, nfsv3 ownership for nfsv4
4. reboot freenas

## Add NFS mnt to RancherOS
```
sudo ros config set mounts '[["10.68.69.2:/mnt/myVol/appsNFS", "/mnt/appsNFS", "nfs4",""]]'
```
## Install Portainer with NFS share
```
sudo docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock --restart always --name portainer -v /mnt/appsNFS/portainer:/data portainer/portainer
```
1. Navigate to http://hostIP:9000 and select local
2. When adding volumes to a container select bind and use the path /mnt/appsNFS/whateveryouwanthere

## Add macvlan so containers are given an IP and mac from your LAN
https://www.portainer.io/2018/09/using-macvlan-portainer-io/

1. click add network
2. select macvlan
3. enter in your lan network
4. select enable manual connection
5. when creating a container select the network you just added and give it an availble static IP

## Rancher OS commands and resources
```
sudo vi /var/lib/rancher/conf/cloud-config.yml  ##edit config file
```
https://medium.com/the-code-review/clean-out-your-docker-images-containers-and-volumes-with-single-commands-b8e38253c271
https://www.digitalocean.com/community/tutorials/how-to-remove-docker-images-containers-and-volumes

# Updates
## Proxmox
Your PVE GUI and slect the upgrade button
## Rancher OS
sudo ros os upgrade
## Update Portainer
```
docker stop portainer
docker rm portainer
docker pull portainer/portainer:latest
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock --restart always --name portainer -v /mnt/appsNFS/portainer:/data portainer/portainer
```
## Install Rancher (optional)

- sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher

- log in to ranhcer thorugh the web browser
- Add Cluster.
- Choose Custom.
- Enter a Cluster Name. Click Next.
- From Node Role, select all the roles: etcd, Control, and Worker.
- Copy the command displayed on screen to your clipboard.
- Log in to your Rancher host with PuTTy. Run the command copied to your clipboard.
- When you finish running the command on your Linux host, click Done.
- Wait for your cluster to finish provisioning
- Reboot to make sure everything is working right

Creating your first container
  - In your cluster drop down tab select default then deploy
  - give it a name and add the ports and env needed
