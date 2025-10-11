# ansible-home-edu

These are my Ansible scripts for an at-home machine primarily used for educational purposes.

## Initial setup

Create a bootable Linux Mint disk.

From there, we will manually configure disks before doing the OS install.

### Disk configuration

I have a 1TB consumer grade NVMe drive for the boot partition and OS files,
and a refurbished 1.92TB enterprise grade SSD for home directories and some other shared files.

The GUI for Linux Mint doesn't let you set up LVM,
but we want LVM to easily manage logical volumes on the SSD.
To set up LVM, then, we have to drop to the terminal and run a sequence of commands,
something like this:

```sh
# Create physical volume.
sudo pvcreate /dev/sda

# Create volume group.
sudo vgcreate vg-data /dev/sda

# Create logical volumes.
sudo lvcreate -L 1300G -n lv-home vg-data
sudo lvcreate -L 100G -n lv-var vg-data
sudo lvcreate -L 300G -n lv-shared vg-data
sudo lvcreate -L 64G -n lv-swap vg-data

# Format the volumes.
sudo mkfs.ext4 /dev/vg-data/lv-home
sudo mkfs.ext4 /dev/vg-data/lv-var
sudo mkfs.ext4 /dev/vg-data/lv-shared
sudo mkswap /dev/vg-data/lv-swap
```

Then, since I was already in the terminal anyway,
I went ahead and manually set up the NVMe drive:

```sh
# Create GPT partition table.
sudo parted /dev/nvme0n1 mklabel gpt

# Create EFI partition.
sudo parted /dev/nvme0n1 mkpart primary fat32 1MiB 513MiB
sudo parted /dev/nvme0n1 set 1 esp on
sudo mkfs.fat -F32 /dev/nvme0n1p1

# Create root partition.
sudo parted /dev/nvme0n1 mkpart primary ext4  513MiB 100%
sudo mkfs.ext4 /dev/nvme0n1p2
```

From there, you can use the GUI to set up the manually created partitions.
The SSD can be marked to use /home, /var, and a swap partition;
the "shared" partition will be marked unused and we will set that up later.

The non-boot partition on the NVMe is mounted at /.

### First boot

You probably want to use the update manager to install any relevant updates,
which will likely require a reboot.
(We'll consider this still the first boot after that.)

Next, we'll install Ansible from the PPA,
so that we can run all the ansible scripts in this repo.

```sh
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible

# And install the community.general collection, for installing flatpak apps from ansible.
ansible-galaxy collection install community.general
```

Finally we should be able to run the playbook:

```sh
ansible-playbook -i inventory.ini playbook.yml -v
```
