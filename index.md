
### How to configure Network-Bound Disk Encryption to automatically open encrypted disks.

#### Introduction

From a security viewpoint, it's important to encrypt our sensitive data to make sure it's protected from prying eyes or hackers. Linux Unified Key Setup or **LUKS** is a great tool to accomplish this.

LUKS is a common standard for Linux disk encryption and stores all pertinent setup information in the partition header which makes migrating data easy.

To configure the encrypted disks or partitions with LUKS you will need to use the **cryptsetup** utility.

Unfortunately one of the downsides to encrypting your disks is manually providing the password every time the system is rebooted or when the disk is remounted. 

Thankfully we have the Network-Bound Disk Encryption or **NBDE** technology to automatically and securely unlock our encrypted disks without any user intervention. 

NBDE is available starting with Red Hat Enterprise Linux 7.4,  CentOS 7.4, and Fedora 24. 

NBDE is implemented using the following technologies:

- Clevis framework 
- Tang server

The Clevis framework is a pluggable framework tool that automatically decrypts and unlocks LUKS volumes.

The Tang server provides the encryption keys to the Clevis client. Tang is a service for binding cryptographic keys to network presence. It offers a secure, stateless, anonymous alternative to key escrow services.

Because NBDE uses the client-server architecture we need to configure both the client and the server.

You can use a virtual machine on your local network for your Tang server.

### Server installation

Install tang using sudo:

~~~
sudo yum install tang -y
~~~

Enable the Tang server:

~~~
sudo systemctl enable tangd.socket --now
~~~

The Tang server works on port 80 and needs to be added to firewalld.

Add the appropriate firewalld rule: 

~~~
sudo  firewall-cmd --add-port=tcp/80 --perm
sudo firewall-cmd --reload
~~~

This completes the server installation.

### Client installation

For this article we'll assume we have a  new 1 GB disk named /dev/vdc that has been added to our system. 

Create the primary partition using `fdisk` or `parted`.

~~~
sudo fdisk /dev/vdc
~~~

Complete the following:

~~~
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x4a6812d4.

Command (m for help): 
~~~

Type **`n`** to create the new partition.

~~~
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended   
Select (default p):
~~~ 

Hit the **`Enter`** key to select primary partition

~~~
Using default response p
Partition number (1-4, default 1):
~~~ 

Hit the **`Enter`** key to select the default partition number.

~~~
First sector (2048-2097151, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-2097151, default 2097151):
~~~

Hit the **`Enter`** key to select the last sector.

~~~
Using default value 2097151
Partition 1 of type Linux and of size 1023 MiB is set

Command (m for help): wq
~~~

Type **`wq`** to save the changes and exit fdisk.

~~~
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
~~~

Run `partprobe` to inform the system of the partition table changes.

~~~
sudo partprobe
~~~

Install the **cryptsetup** package using sudo:

~~~
sudo yum install cryptsetup -y
~~~

Use the **cryptsetup luksFormat** command to encrypt the disk. You will need to type **`YES`** when prompted and also to choose and enter a passphrase to encrypt the disk.

~~~
sudo cryptsetup luksFormat /dev/vdc1
WARNING!
========
This will overwrite data on /dev/vdc1 irrevocably.

Are you sure? (Type uppercase yes):

Enter passphrase for /dev/vdc1: 
Verify passphrase:
~~~

Use the **cryptsetup luksOpen** command to map the encrypted partition to a logical device. We'll use `encryptedvdc1` as the name. You will also need to enter the passphrase again.

~~~
sudo cryptsetup luksOpen /dev/vdc1 encryptedvdc1
Enter passphrase for /dev/vdc1: 
~~~

The encrypted partition is now available at `/dev/mapper/encryptedvdc1`.

Create an XFS filesystem on the encrypted partition.

~~~
sudo mkfs.xfs /dev/mapper/encryptedvdc1
~~~

Create a directory for mounting the encrypted partition:

~~~
sudo mkdir /encrypted
~~~

Use the **cryptsetup luksClose** command to lock the partition.

~~~
cryptsetup luksClose encryptedvdc1
~~~

Install the Clevis packages using sudo:

~~~
sudo yum install clevis clevis-luks clevis-dracut -y
~~~

Modify the `/etc/crypttab` file to open the encrypted partition at boot time.

~~~
sudo vim /etc/crypttab
~~~

Add the following line:

~~~
encryptedvdc1       /dev/vdc1  none   _netdev
~~~

Modify the `/etc/fstab` file to mount the encrypted volume during reboot.

~~~
sudo vim /etc/fstab
~~~

Add the following line:

~~~
/dev/mapper/encryptedvdc1   /encrypted       xfs    _netdev        1 2
~~~

For this article we'll assume the IP address of the Tang server is `192.168.1.20`.  You can also use the host name or domain if you prefer.

Run the following `clevis` command:

~~~
sudo clevis bind luks -d /dev/vdc1 tang '{"url":"http://192.168.1.20"}'
The advertisement contains the following signing keys:

rwA2BAITfYLuyNiIeYUMBzkhk7M

Do you wish to trust these keys? [ynYN] Y
Enter existing LUKS password: 
~~~

You will have to type **`Y`** to accept the keys for the Tang server and provide the existing LUKS password for the initial setup. 

Enable `clevis-luks-askpass.path` to support non-root LUKS-encrypted partitions.

~~~
sudo systemctl enable clevis-luks-askpass.path
~~~

This completes the client installation.

Now whenever you reboot the server, the encrypted disk should automatically be decrypted and mounted by retrieving the keys from the Tang server.

If the Tang server is unavailable for any reason, you'll need to provide the passphrase manually in order to decrypt and mount the partition.
