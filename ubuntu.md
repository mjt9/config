
# Ubuntu 24.04

## Bug in current install
Get a warning at boot:
**WARNING: Option 'size' missing in crypttab for plain dm-crypt mapping root

Check what values to put in crypttab
```
sudo cryptsetup status dm_crypt-0
```

Then the line in crypttab should be like the following:
```
dm_crypt-0 PARTUUID=... /dev/urandom swap,initramfs,plain,offset=0,cipher=aes-cbc-essiv:sha256,size=256
```

Then enable
```
sudo swapoff -a
sudo update-initramfs -c -k all
sudo swapon -a
```

Note: There still could be a warning saying 
**WARNING: Resume target dm_crypt-0 uses a key file

This is likely a false positive because the initramfs option in crypttab
enables the disk to open by initramfs, early in the boot when a key file
might not be available, which is not the case for /dev/random.
The initramfs option doesn't *seem* to make it a Resume target.

## Opt out of beta testing updates

Create the file 
```
sudo vim /etc/apt/apt.conf.d/99-Phased-Updates
```

And enter the following lines:
```
Update-Manager::Never-Include-Phased-Updates true;
APT::Get::Never-Include-Phased-Updates true;
```

## Disabling auto-suspend including in GDM

```
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
sudo apt install dbus-x11
xhost + local:
sudo -u gdm dbus-launch gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
xhost - local:
```

## C++ Development

```
sudo apt install clangd clang-format gcc-14 g++-14
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-14 100                     \
                         --slave   /usr/bin/g++ g++ /usr/bin/g++-14                         \
                         --slave   /usr/bin/gcov gcov /usr/bin/gcov-14                      \
                         --slave   /usr/bin/gcov-dump gcov-dump /usr/bin/gcov-dump-14       \
                         --slave   /usr/bin/gcov-tool gcov-tool /usr/bin/gcov-tool-14       \
                         --slave   /usr/bin/gcc-ar gcc-ar /usr/bin/gcc-ar-14                \
                         --slave   /usr/bin/gcc-nm gcc-nm /usr/bin/gcc-nm-14                \
                         --slave   /usr/bin/gcc-ranlib gcc-ranlib /usr/bin/gcc-ranlib-14
sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/g++ 100
sudo update-alternatives --install /usr/bin/cc cc /usr/bin/gcc 100
```

## Emacs configuration

```
sudo apt install --no-install-recommends emacs elpa-lsp-ui elpa-lsp-mode elpa-lsp-treemacs elpa-treemacs elpa-which-key elpa-company
```

Copy `init.el` to `~/emacs.d/`.

The lsp-treemacs and company packages are currently missing icons. These can be downloaded from:
```
https://github.com/emacs-lsp/lsp-treemacs/tree/0.4
https://github.com/company-mode/company-mode/tree/0.10.0
```
Then copied, e.g.
```
sudo cp -r lsp-treemacs-0.4/icons  /usr/share/emacs/site-lisp/elpa/lsp-treemacs-0.4/
sudo cp -r company-mode-0.10.0/icons  /usr/share/emacs/site-lisp/elpa/company-0.10.0/
```


## Encrypted ZFS

The root pool is create by the installer with the following command.

```
History for 'rpool':
2024-03-22.21:07:06 zpool create -o ashift=12 -o autotrim=on -O canmount=off -O normalization=formD -O acltype=posixacl -O compression=lz4 -O devices=off -O dnodesize=auto -O relatime=on -O sync=standard -O xattr=sa -O encryption=on -O keylocation=file:///tmp/tmpp7696c3m -O keyformat=raw -O mountpoint=/ -R /target rpool /dev/disk/by-id/ata-CIE_MS_M335_128GB_204600000193-part4
2024-03-22.21:07:06 zpool set cachefile=/etc/zfs/zpool.cache rpool
2024-03-22.21:07:06 zfs create -o encryption=off -V 20971520 rpool/keystore
2024-03-22.21:07:14 zfs set keylocation=file:///run/keystore/rpool/system.key rpool
2024-03-22.21:07:14 zfs create -o canmount=off -o mountpoint=none rpool/ROOT
2024-03-22.21:07:14 zfs create -o canmount=on -o mountpoint=/ rpool/ROOT/ubuntu_omybwd
2024-03-22.21:07:14 zfs create -o canmount=off rpool/ROOT/ubuntu_omybwd/var
........
```

These are adjusted for additional encrypted ZFS disks. The files in ```/etc/zfs/zfs-list.cache/``` are created first, so that when any changes are seen to those pools by a daemon, the files are updated.

```
mkdir /etc/zfs/zfs-list.cache
touch /etc/zfs/zfs-list.cache/librarium
touch /etc/zfs/zfs-list.cache/reclusiam

zpool create -o ashift=12 -o autotrim=on -O normalization=formD -O acltype=posixacl -O compression=lz4 -O dnodesize=auto -O relatime=on -O sync=standard -O xattr=sa -O encryption=on -O keylocation=file:///run/keystore/rpool/system.key -O keyformat=raw librarium /dev/disk/by-id/ata-Micron_5100_MTFDDAK1T9TCB_1721174CDF87

zpool create -o ashift=12 -o autotrim=on -O normalization=formD -O acltype=posixacl -O compression=lz4 -O dnodesize=auto -O relatime=on -O sync=standard -O xattr=sa -O encryption=on -O keylocation=file:///run/keystore/rpool/system.key -O keyformat=raw reclusiam /dev/disk/by-id/ata-MTFDDAK3T8TCB_18111BB81335

zfs create librarium/docker
```

Check that the cache files have been updated
```
cat /etc/zfs/zfs-list.cache/librarium
```

Back up the keystore to the encrypted disks, make sure the key is not encrypted with the key!
```
zfs snapshot rpool/keystore@snap1
zfs send rpool/keystore@snap1 | zfs receive -o encryption=off reclusiam/keystore
zfs send rpool/keystore@snap1 | zfs receive -o encryption=off reclusiam/keystore
```

The LUKS device should be accessible here
```
cryptsetup luksDump /dev/zvol/librarium/keystore
```

Check which data is encrypted
```
zfs list -o name,encryption
```

## Docker root directory

Create/edit the file '''/etc/docker/daemon.json'''  to add

```
{
  "data-root":"/librarium/docker"
}
```


## ZFS encryption with passphrase

Options taken from above, but with a prompt for a passphrase instead of a keyfile.

```
sudo zpool create -o ashift=12 -o autotrim=on -O normalization=formD -O acltype=posixacl -O compression=lz4 -O dnodesize=auto -O relatime=on -O sync=standard -O xattr=sa -O encryption=on -O keylocation=prompt -O keyformat=passphrase citadel /dev/disk/by-id/usb-Seagate_One_Touch_SSD_00000000NAE70KS6-0:0
```

Mount using

```
sudo zfs load-key citadel
sudo zfs mount citadel/reclusiam
```


## UMR
```
sudo apt install libpciaccess-dev libncurses-dev libdrm-dev llvm-18 libnanomsg-dev libjson-c-dev libsdl2-dev
git clone https://gitlab.freedesktop.org/tomstdenis/umr
```
