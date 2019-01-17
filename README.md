# Freeside Dev Environment with VirtualBox and NFS
I have come to prefer this as an alternative to
[freeside-vagrant](https://github.com/mitchjacksontech/freeside-vagrant).
This approach is not as automated.  Instead of creating new VMs  as needed with
a vagrant build process, vm versioning/cloning is managed by hand with
VirtualBox.

This is more of a guide, and less of a functioning tool, than
[freeside-vagrant](https://github.com/mitchjacksontech/freeside-vagrant).

## Summary

This approach provides
* Simple boot-up/shut-down of various VMs
* Directories from the Guest VM are mouted to the Host file system via NFS.
  This allows a native code editor on the host to work with code on the guest

## Set-UP

### 1. Create Freeside VMs with Virtualbox

At the moment, I create six VMs with Virtualbox, with the following names:
* freeside-deb9-fsm - Debian 9, Freeside master branch
* freeside-deb9-fs4 - Debian 9, Freeside v4 branch
* freeside-deb9-fs3 - Debian 9, Freeside v3 branch
* freeside-deb8-fsm - Debian 8, Freeside master branch
* freeside-deb8-fs4 - Debian 8, Freeside v4 branch
* freeside-deb8-fs3 - Debian 8, Freeside v3 branch

Clone the Freeside repository into /usr/local/src/freeside

### 2. Configure Networking for VMs

Add two network adapters to the VM
* Host-Only Adapter - For local traffic
* Bridged Network adapter - For internet traffic

Edit the Host machine's hosts file.  On the mac, this is found
at `/private/etc/hosts` or on linux, `/etc/hosts`.  Add a hosts
entry for each virtual machine, to match the VirtualBox VM names.
Use the IP address assigned to each of these VMs

``` conf
# /etc/hosts or /private/etc/hosts
10.0.0.11 freeside-deb9-master
10.0.0.12 freeside-deb9-fs4
10.0.0.13 freeside-deb9-fs3
10.0.0.14 freeside-deb8-master
10.0.0.15 freeside-deb8-fs4
10.0.0.16 freeside-deb8-fs4

```

At this point, you should be able to run `ssh freeside-deb9-master` and
reach your virtual machine from the host.

### 3. Configure NFS Exports within VMs

Note:
This is not a secure NFS configuration - any machine that can connect to
the VM can get full access to these NFS shares.  If your network is configured
correctly, the only machine able to access the VM is the Host machine

Install nfs server on the Guest VM: `apt install nfs-server`

Add entries to the exports file

``` conf
/var/www/html *(rw,sync,no_wdelay,no_subtree_check,all_squash,anonuid=1001,anongid=1001)
/usr/local/src/freeside *(rw,sync,no_wdelay,no_subtree_check,all_squash,anonuid=0,anongid=0)
/usr/local/share/perl/5.24.1 *(rw,sync,no_wdelay,no_subtree_check,all_squash,anonuid=0,anongid=0)

```

* /usr/local/src/freeside - Allows editing of the source files contained
  within the git repository
* /var/www/html - Allows editing of the deployed mason templates in use
  by apacahe
* /usr/local/share/perl/5.24.1 - Allows editing of the deployed perl libraries
  in use on the system by apache and scripts

### 4. Configure a shared folder between all VMs

Open VirtualBox Manager.  In the settings for each VM, add a Shared
Folder:
* Name: common_tools
* Path
  * Linux host: /home/username/.freeside/common_tools
  * MacOS host: /Users/username/.freeside/common_tools
* Auto-mount: Yes
* Access: Full

### 5. Configure NFS mounts on Host OS

Edit the Host OS fstab file, and add entries for each mount for each vm.
Replace /home/username with your correct home directory location

``` conf
# /etc/fstab or /private/etc/fstab
freeside-deb9-master:/var/www/html /home/username/.freeside/f9m/www nfs rw,noauto,user 0 0
freeside-deb9-master:/usr/local/src/freeside /home/username/.freeside/f9m/src nfs rw,noauto,user 0 0
freeside-deb9-master:/usr/local/share/perl/5.24.1 /home/username/.freeside/f9m/perlib nfs rw,noauto,user 0 0

freeside-deb9-fs4:/var/www/html /home/username/.freeside/f94/www nfs rw,noauto,user 0 0
freeside-deb9-fs4:/usr/local/src/freeside /home/username/.freeside/f94/src nfs rw,noauto,user 0 0
freeside-deb9-fs4:/usr/local/share/perl/5.24.1 /home/username/.freeside/f94/perlib nfs rw,noauto,user 0 0

freeside-deb9-fs3:/var/www/html /home/username/.freeside/f93/www nfs rw,noauto,user 0 0
freeside-deb9-fs3:/usr/local/src/freeside /home/username/.freeside/f93/src nfs rw,noauto,user 0 0
freeside-deb9-fs3:/usr/local/share/perl/5.24.1 /home/username/.freeside/f93/perlib nfs rw,noauto,user 0 0

```

### 6. Create directories for local system mount points
Of course, replace /home/username with the correct location for your system
```
mkdir -p /home/username/.freeside/f9m/src
mkdir -p /home/username/.freeside/f9m/perlib
mkdir -p /home/username/.freeside/f9m/www

mkdir -p /home/username/.freeside/f94/src
mkdir -p /home/username/.freeside/f94/perlib
mkdir -p /home/username/.freeside/f94/www

mkdir -p /home/username/.freeside/f93/src
mkdir -p /home/username/.freeside/f93/perlib
mkdir -p /home/username/.freeside/f93/www
chmod ug+rwx -R /home/username/.freeside
```

### 7. Individual commands

These commands can be issued at the Host shell

* Boot: `VBoxHeadless -s "freeside-deb9-master" &`
* Shut down: `VBoxManage controlvm "freeside-deb9-master" acpipowerbutton`
* SSH: ssh freeside-deb9-master
* Mount source directories:
  ```
  cd /home/username/.freeside/f9m
  mount src
  mount perlib
  mount www
  ```
* Unmount source directories:
  ```
  cd /home/username/.freeside/f9m
  umount src
  umount perlib
  umount www
  ```

### 8. Shell utility script to put it all together

See [this utility script](vm). I use this script to manage both
Vagrant VMs and these freeside VirtualBox VMs.  Install the script
in your ~/bin folder, and then use the commands:

* Start VM, mount exports, and SSH into it: `vm f9m up`
* SSH into vm: `vm f9m`
* Unmount exports, shutdown VM: `vm f9m down`