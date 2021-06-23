Right so let's begin, I started this "guide" of sorts since I'm currently in a household of limited bandwidth and 4 heavy bandwidth users, 90% of this consists of just Steam, uPlay or Origin games, the original guide didn't make much sense to me as it was far too technicial and required too many links to jump to and forth, so I'm trying to simple it down here, this is assuming you have a basic knowledge of linux and if you don't, that you have the ability to google.

So this lancache is super duper handy if you are hosting a lan party or if the people in your house / area of server are downloading games a lot, this cuts tons of bandwidth, Mine kept up pretty well during a 50~ person lan party.



## Things You Need

  - Some form of linux, I prefer Ubuntu
  - At least 8GB of RAM.
  - A decent-ish processor
  - A 2 Terabyte Drive
  
  

## What I have
  - a i5-3570
  - 32 GB of DDR3
  - a few drives in a ZFS Raid0 pool (2x 1TB mechanical + 2x 500gb mechanical, ZFS is hosted on the proxmox, the VM only sees one additional drive to the boot drive)
  - a Proxmox Hypervisor for Virtual Machines
  
## Start Up Instuctions
  
- Start up VM or machine (I'm using Ubuntu Server / CLI 18.04)
- Setup hostname / IP Address Settings (I prefer to do a static lease from the router, but that's just me)
- Install QEMU-Agent for Proxmox - ```apt-get install qemu-guest-agent``` (Skip if not using Proxmox)
- use `lsblk` to find out what extra drive is going to be used to store the cache - note the drive (for my example, it's SDB)
- We're using fdisk to partiiton it - `fdisk /dev/sdb` - type g <enter> then n <enter> <enter> <enter> p <enter>(at this point, it should say /dev/sdb1) w <enter>
- a quick check of `lsblk` will confirm that the drive has been partitioned as sdb1
- Now we're going to make it ext4 - `mkfs.ext4 /dev/sdb1` (this may take a while to write)
- make the directory for the drive to mount to `mkdir /CACHE` 
- Now we're going to the fstab to mount it on boot `nano /etc/fstab` - copy and paste this next part in after creating a new line
- ``` /dev/sdb1       /CACHE    ext4    defaults        0       0```
- hit control and o <enter>, hit control and x
- now we're going to mount it - `sudo mount -a`

## DOCKER (START HERE if you have already set up a basic machine and are not using a dedicated drive for the cache)
- This is legit copy pasted from - https://docs.docker.com/install/linux/docker-ce/ubuntu/
- Make sure any previous versions of Docker aren't installed by using 
 - ```sudo apt-get remove docker docker-engine docker.io containerd runc```
 - Basic update APT ```sudo apt-get update```
 - install packages to allow apt to use a repository over HTTPS: 
 - ```sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common```
 - choose yes for everything
 - Add Docker’s official GPG key: 
 ```curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -```
 - Use the following command to set up the stable repository. 
 ```sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   ```
 - Update APT again ```sudo apt-get update```
 - Install the latest version of Docker Engine - Community and containerd
 ```sudo apt-get install docker-ce docker-ce-cli containerd.io```
 - Verify that Docker Engine - Community is installed correctly by running the hello-world image.
 ```sudo docker run hello-world```
 
 ## Now that Docker's installed
- Let's quickly make some changes so that the script coming up won't screw things up
- head to /etc/hosts and copy your hostname to the lists under 127.0.0.1
- `sudo nano /etc/hosts` 
- make a new line under "127.0.0.1  localhost.localdomain localhost" and add the following
- ```127.0.0.1 <HOSTNAME>```
- hit control and o <enter> and then control and x
- now we are going to restart the machine
- `sudo reboot`
- then we're going to disable systemd-resolve
- `sudo systemctl stop systemd-resolve`
- `sudo systemctl disable systemd-resolve`

- head to the /CACHE and make two folders, "data" and "logs"
- ```cd /CACHE```
- ```mkdir data```
- ```mkdir logs```
- make a script called "update.sh"
  ``` sudo nano update.sh```
- copy paste the next part in with changes to the parts with < >
  
```
sudo docker stop sniproxy
sudo docker stop lancache
sudo docker stop steamcache-dns
sudo docker rm sniproxy
sudo docker rm lancache
sudo docker rm steamcache-dns
sudo docker run --restart unless-stopped --name lancache --detach -v /CACHE/data:/data/cache -v /CACHE/logs:/data/logs -p 80:80 -e CACHE_MEM_SIZE=<HOW MUCH RAM>m -e CACHE_DISK_SIZE=1900g steamcache/monolithic:latest
sudo docker run --restart unless-stopped --name sniproxy --detach -p 443:443 steamcache/sniproxy:latest
sudo docker run --restart unless-stopped --name steamcache-dns --detach -p 53:53/udp -e UPSTREAM_DNS="1.1.1.1" -e USE_GENERIC_CACHE=true -e LANCACHE_IP="x.x.x.x" steamcache/steamcache-dns:latest
```

- You will need to change CACHE_MEM_SIZE to how much ram you want to use, for my example, I used 16384m which is 16GB
- The CACHE_DISK_SIZE=1900g is assuming you have a 2TB drive used as the additional storage drive, but change this accordingly
- UPSTREAM_DNS="1.1.1.1" is assuming you want to use 1.1.1.1 as your dns provider, you can change this accordingly
- LANCACHE_IP="x.x.x.x" - You need to change this to whatever IP address your machine has

- Now everything should be running, you can point your router's DNS towards the machine's IP address and the cache should work!
- but what if the machine's power goes off? Let's make a script and add it to systemd so that the cache starts on boot!
```cd /CACHE```
```nano start.sh```
- copy paste the next part
```
sudo docker stop sniproxy
sudo docker stop lancache
sudo docker stop steamcache-dns
sudo docker start sniproxy
sudo docker start lancache
sudo docker start steamcache-dns
```
- make sure to add the +x permissions to start.sh
```sudo chmod +x start.sh
```
- Now we have to create the systemd service and link it
- ```sudo nano /lib/systemd/system/cache.service```
- Copy paste the next part to the file
```[Unit]
Description=monolithic

[Service]
Type=oneshot
ExecStart=/bin/bash /CACHE/start.sh
RemainAfterExit=true
StandardOutput=journal

[Install]
WantedBy=multi-user.target
```
- Hit control and O <enter> and then control and x
- enable the service 
```sudo systemctl enable cache```
```sudo systemctl start cache
```

- and there you have it! Now we are done!!

- I hope everything works, if it doesn't, then please let me know
- otherwise, I'll fix up the formatting a bit later on


  
  
