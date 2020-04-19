#### SYSTEM DESIGN

##### Disk partitioning  "GPT"
```
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                   8:0    0 465.8G  0 disk
|-sda1                8:1    0   512M  0 part  /boot/efi
|-sda2                8:2    0   514M  0 part
| `-encrypted-boot  253:5    0   512M  0 crypt /boot
`-sda3                8:3    0 464.8G  0 part
  `-root            253:2    0 464.8G  0 crypt
    |-notebook-root 253:3    0   100G  0 lvm   /
    `-notebook-home 253:4    0   200G  0 lvm   /home
```

##### FSTAB --> directories stored in memory V2
```
UUID=586fbba9-d38d-43c1-9e8b-b780a9d7a76d   /                        ext4   rw,relatime,data=ordered 0 1
UUID=d8546367-f2ee-46a4-b587-b926703bfdb1   /home                    ext4   rw,relatime,data=ordered 0 0
UUID=4fa59ac1-df34-45d1-b91a-2f37ca0c2f6b   /boot                    ext4   noauto,rw,relatime,data=ordered 0 0
UUID=2E77-3F97                              /boot/efi                vfat   noauto,rw 0 0

tmpfs            /tmp               tmpfs    size=4G                               0 0
tmpfs            /run               tmpfs    size=100M                             0 0
tmpfs            /var/tmp           tmpfs    rw,nosuid,noatime,nodev,size=4G,mode=1777  0 0
tmpfs            /var/tmp/portage   tmpfs    rw,nosuid,noatime,nodev,size=8G,mode=755,uid=portage,gid=portage,x-mount.mkdir=755  0 0

tmpfs            /var/log           tmpfs    defaults,noatime,nosuid,mode=0755,size=100m    0 0
tmpfs            /var/run           tmpfs    defaults,noatime,nosuid,mode=0755,size=2m    0 0
tmpfs            /var/spool/mqueue  tmpfs    defaults,noatime,nosuid,mode=0700,gid=12,size=30m    0 0

shm              /dev/shm           tmpfs    nodev,nosuid,noexec                   0 0
proc             /proc              proc     rw,nosuid,nodev,noexec,relatime       0 0
```


#### BUILD KERNEL

##### Mount boot/efi 
1. decrypted boot 
```
cryptsetup open /dev/sda2 encrypted_boot
```
2. mount boot 
```
mount /dev/mapper/encrypted_boot /boot/
```
3. mount efi
```
mount /dev/sda1  /boot/efi/
```

##### V1 Self assembly kernel/initramfs
0. ``` cd /usr/src/linux/ ```

1. build kernel
```
make -j9 && make -j9 modules_install && make install
```
2. build initramfs
```
genkernel --lvm --luks initramfs
```
3. updte grub.cfg
```
grub-mkconfig -o /boot/grub/grub.cfg
```

##### V2 genkernel build, initramfs integrated in kernel
```
genkernel --kernel-config=/proc/config.gz --no-clean --integrated-initramfs --kernel-append-localversion=-test1  --lvm --luks all
```

> ps. genkernel при сборке ядра модифицирует default .config --> /usr/src/linux/.config

##### fix stable kernel
when you will be get stable kernel you need make stable copy, for recovery/current boot

```
cp -v System.map-4.12.12-gentoo System.map-4.12.12-gentoo-stable
cp -v config-4.12.12-gentoo     config-4.12.12-gentoo-stable
cp -v initramfs-4.12.12-gentoo  initramfs-genkernel-x86_64-4.12.12-gentoo-stable
cp -v vmlinuz-4.12.12-gentoo    vmlinuz-4.12.12-gentoo-stable
```

and fix grub.conf with your hands

#### SECURE_BOOT
```
emerge -av app-crypt/efitools
```

The UEFI specification defines four secure, non-volatile variables, which are used to control the secure boot subsystem. They are:

1. The **Platform Key (PK)**. The PK variable contains a UEFI (small 's', small 'd') 'signature database' which has at most one entry in it. When PK is emptied (which the user can perform via a BIOS GUI action), the system enters setup mode (and secure boot is turned off). In setup mode, any of the four special variables can be updated without authentication checks. However, immediately a valid platform key is written into PK (in practice, this would be an X.509 public key, using a 2048-bit RSA scheme), the system (aka, 'platform') enters user mode. Once in user mode, updates to any of the four variables must be digitally signed with an acceptable key. The private key counterpart to the public key stored in PK may be used to sign user-mode updates to PK or KEK, but not db or dbx (nor can it be used to sign executables).
2. The **Key Exchange Key (KEK)**. This variable holds a signature database containing one (or more) X.509 / 2048-bit RSA public keys (other formats are possible). In user mode, any db/dbx (see below) updates must be signed by the private key counterpart of one of these keys (the PK cannot be used for this). While KEK keys (or, more accurately, their private-key counterparts) may also be used to sign executables, it is uncommon to do so, since that's really what db is for (see below).
3. The (caps 'S', caps 'D') **Signature Database (db)**. As the name suggests, this variable holds a UEFI signature database which may contain (any mixture of) public keys, signatures and plain hashes. In practice, X.509 / RSA-2048 public keys are most common. It functions essentially as a boot executable whitelist (described in more detail shortly).
4. The **Forbidden Signatures Database (dbx)**. This variable holds a signature database of similar format to db. It functions essentially as a boot executable blacklist.

##### Preparation
```
mkdir /etc/efikeys
cd /etc/efikeys
```

##### Generate self sign certs with keys for Secure Boot: PK, KEK, db
```
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=corealugly's platform key/" -keyout PK.key -out PK.cert.pem -days 3650 -nodes -sha256
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=corealugly's key-exchange-key/" -keyout KEK.key -out KEK.cert.pem -days 3650 -nodes -sha256
openssl req -new -x509 -newkey rsa:2048 -subj "/CN=corealugly's kernel-signing key/" -keyout db.key -out db.cert.pem -days 3650 -nodes -sha256
```
```
chmod -v 400 *.key
```

##### Backup old key
```
efi-readvar -v PK -o old_PK.esl
efi-readvar -v KEK -o old_KEK.esl
efi-readvar -v db -o old_db.esl
efi-readvar -v dbx -o old_dbx.esl
```

##### Convert open part of the keys to the ESL format understood for UEFI
```
cert-to-efi-sig-list -g "$(uuidgen)" PK.cert.pem PK.esl
cert-to-efi-sig-list -g "$(uuidgen)" KEK.cert.pem KEK.esl
cert-to-efi-sig-list -g "$(uuidgen)" db.cert.pem db.esl
```

##### Sign ESL files
```
sign-efi-sig-list -k PK.key -c PK.cert.pem PK PK.esl PK.auth
sign-efi-sig-list -a -k PK.key -c PK.cert.pem KEK KEK.esl KEK.auth
sign-efi-sig-list -a -k KEK.key -c KEK.cert.pem db db.esl db.auth

sign-efi-sig-list -k KEK.key -c KEK.cert dbx old_dbx.esl old_dbx.auth
```

##### Clear ALL UEFI key
- need to enter the uefi and "Delete All Keys" 
- check: ```efi-readvar```

##### Create DER versions of each of our three new public keys
```
openssl x509 -outform DER -in PK.cert.pem -out PK.cert.der
openssl x509 -outform DER -in KEK.cert.pem -out KEK.cert.der
openssl x509 -outform DER -in db.cert.pem -out db.cert.der
```

#### This block only for save windows certificate
##### Create compound (i.e., old+new) esl files for the KEK and db, and also create .auth counterparts for these.
```
cat old_KEK.esl KEK.esl > compound_KEK.esl
cat old_db.esl db.esl > compound_db.esl
sign-efi-sig-list -k PK.key -c PK.cert.pem KEK compound_KEK.esl compound_KEK.auth
sign-efi-sig-list -k KEK.key -c KEK.cert.pem db compound_db.esl compound_db.auth 
```

##### Install new key in keystore
```
efi-updatevar -e -f old_dbx.esl dbx
efi-updatevar -e -f compound_db.esl db
efi-updatevar -e -f compound_KEK.esl KEK

efi-updatevar -f PK.auth PK
```

##### Export/backup certificate 
> if all norm, backup all components
```
efi-readvar -v PK -o new_PK.esl
efi-readvar -v KEK -o new_KEK.esl
efi-readvar -v db -o new_db.esl
efi-readvar -v dbx -o new_dbx.esl
```

##### Sign grubx64.efi
```
sbsign --key /etc/efikeys/db.key  --cert  /etc/efikeys/db.crt  --output  /boot/efi/EFI/corebook/grubx64.efi.signed /boot/efi/EFI/corebook/grubx64.efi
```

##### Add boot row in UEFI
```
efibootmgr -c -L 'gcore.signed' -l '\EFI\corebook\grubx64.efi.signed'
```
