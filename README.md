# Ubuntu lx-brand Image Builder

This is a collection of scripts used for creating an lx-brand Ubuntu image.

## Requirements

In order to use these scripts you'll need:

- Ubuntu (or Debian) running in a VM or bare metal (required for the `install` script)
- debootstrap & git: `apt-get install -y debootstrap git`
- A SmartOS (or SDC headnode) install (required for the `create-lx-image` script)

**Note***: The build scripts currently assume you are running under a KVM ubuntu-certified instance that has a secondary disk mounted to `/mnt`. The scripts have not been tested on an lx-brand instance.


## Usage

1. Run `./install -d <chroot> -m <mirror> -i <image name> -p <proper name> -u <image docs>` under Ubuntu to install Ubuntu 14.04 in a given directory. This will create a tarball of the installation in your working directory (named `<image name>-<YYMMDD>.tar.gz`). See `./install -h` for detailed usage.
2. Copy the tarball to a SmartOS machine or SDC headnode and run `./create-lx-image -t <TARBALL> -i <IMAGE_NAME> -d <DESC> -u <DOCS>` (substituting the name of your tar file). This will create the image file and manifest. See `/create-lx-image -h` for detailed usage.

## Beginners' guide

1. Set up a BHYVE or KVM Ubuntu instance on your SmartOS host. Give it plenty of vCores and RAM for the building process. Also add a second disk which is mounted on /mnt.

Example template:
```bash
{
  "brand": "bhyve",
  "alias": "ubuntu-lx-brand-image-builder",
  "hostname": "ubuntu-lx-brand-image-builder",
  "resolvers": [
    "8.8.8.8",
    "1.1.1.1"
  ],
  "ram": "4096",
  "vcpus": "6",
  "nics": [
    {
      "nic_tag": "igb0",
      "ip": "10.10.30.120",
      "netmask": "255.255.255.0",
      "gateway": "10.10.30.1",
      "model": "virtio",
      "primary": true
    }
  ],
  "disks": [
    {
      "image_uuid": "cb0849d5-d890-4158-b788-07b11718179d",
      "boot": true,
      "model": "virtio"
    },
    {
      "boot": false,
      "model": "virtio",
      "size": 15000
    }
 ],
"customer_metadata": {
 "root_authorized_keys": "pubkey here",
 "user-script" : "/usr/sbin/mdata-get root_authorized_keys > ~root/.ssh/authorized_keys ; /usr/sbin/mdata-get root_authorized_keys > ~admin/.ssh/authorized_keys"  
}
}

```

Log into the Ubuntu VM we've just created.

```zconsole <VM-UUID>```

Install a few missing packages.

```apt-get install -y debootstrap git apt-transport-https```

Clone our repository.

```git clone https://github.com/archfan/ubuntu-lx-brand-image-builder.git && cd ubuntu-lx-brand-image-builder```

Create the chroot directory.

```mkdir /mnt/chroot```

Execute the command to create a lx image - modify it as you see fit.
```
./install -r bionic -a amd64 -d /mnt/chroot -m http://archive.ubuntu.com/ubuntu/ -i lx-ubuntu-18.04-archfan -p "Archfan's Ubuntu 18.04 LX Brand" -D "Archfan's Ubuntu 18.04 64-bit lx-brand image." -u https://docs.joyent.com/images/container-native-linux
```
Copy the file over to your SmartOS hypervisor.

```scp lx-ubuntu-18.04-archfan-20190429.tar.gz  root@smartos-server-ip:/opt/lx-images```

Install git and clone the repository again.

```pkgin install git && cd /opt && git clone https://github.com/archfan/ubuntu-lx-brand-image-builder.git && cd ubuntu-lx-brand-image-builder```

Create the image file and manifest.

```/create-lx-image -t /opt/lx-images/lx-ubuntu-18.04-archfan-20190429.tar.gz -k 3.16.65 -m 20190424T233834Z -i lx-ubuntu-18.04 -d "Ubuntu 18.04 lx 64-bit lx-brand image." -u https://docs.joyent.com/images/container-native-linux```

Install the image with imgadm.

```imgadm install -m lx-ubuntu-18.04-20190429.json -f lx-ubuntu-18.04-20190429.zfs.gz```

Done.
