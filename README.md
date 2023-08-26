:o: **iSCSI – NAS**

:link:[Tutorial](https://www.howtoforge.com/tutorial/how-to-setup-iscsi-storage-server-on-ubuntu-2004-lts/)
:link:[Tutorial](https://manpages.ubuntu.com/manpages/xenial/man8/tgtadm.8.html)

:o: **Installation:**

:one: **Target:** Server there all disks ready to mount to initiator Server virtually.
:two: **Initiator:** Server will communicate with target.

:arrow_right: **Install iSCSI Target**
```R
apt-get install tgt -y
systemctl status tgt
tgt-admin -h
```

:arrow_right: **Configure iSCSI Target**

```R

root@nas:/etc/tgt/conf.d# cat md1.conf 
<target iqn.2023-05.bioturing.com:md1>
     # Provided device as an iSCSI target
     backing-store /dev/md1
     initiator-address 192.168.0.20
     #initiator-name iqn.2023-05.bioturing.com:nas.md1
     #incominguser bioturing nasB2050
     #outgoinguser bioturing nasB2050
</target>
```
:arrow_right: **Delete specific target**

```R
root@iscsi-target:/mnt# tgtadm --lld iscsi --op delete --force --mode target --tid 4
root@iscsi-target:/mnt# tgtadm --lld iscsi --op show --mode target
```

:arrow_right: **Add Target**

```R
root@iscsi-target:/etc/tgt/conf.d# cat de.conf
<target iqn.2023-06.lalit.com:de>
     # Provided device as an iSCSI target
     backing-store /dev/xvde
     initiator-address 172.31.4.196
</target>
root@iscsi-target:/etc/tgt/conf.d# tgtadm --lld iscsi --op new --mode target --tid 2 -T iqn.2023-06.lalit.com:de
root@iscsi-target:/etc/tgt/conf.d#
```

:arrow_right: **Add logical volume unit.**

```R
root@iscsi-target:/etc/tgt/conf.d# tgtadm --lld iscsi --op new --mode logicalunit --tid 2 --lun 1 -b /dev/xvde
root@iscsi-target:/etc/tgt/conf.d#  tgtadm --lld iscsi --op show --mode target
```

:arrow_right: **Delete logical volume unit.**
```R
root@iscsi-target:/mnt#  tgtadm --lld iscsi --mode logicalunit --op delete --tid=2 --lun=1
```

:arrow_right: **View target defined.**

```R
root@iscsi-target:/mnt# tgtadm --lld iscsi --op show --mode target
```

:arrow_right: **Enable the target with tid 1 in all the interfaces.**

```R
root@iscsi-target:/mnt# tgtadm --lld iscsi --op bind --mode target --tid 1 -I ALL
```

:arrow_right: **Update recovery from Config file.**
```R
root@iscsi-target:/mnt# tgt-admin --update tid=4 -v
```
:bell: **default-driver not defined, defaulting to iscsi.**

:arrow_right: **Target with tid 4 (iqn.2023-06.lalit.com:de) is in use, it won't be updated.**
```R
# default-driver not defined, defaulting to iscsi.
root@iscsi-target:/mnt# tgt-admin --update tid=4 -v -f
```

:arrow_right: **Removing target: iqn.2023-06.lalit.com:de**
```R
tgtadm -C 0 --op unbind --mode target --tid 4 -I 172.31.4.196
tgtadm -C 0 --op delete --mode conn --tid 4 --sid 5 --cid 0
tgtadm -C 0 --mode target --op delete --tid=4
```

:arrow_right: **Adding target: iqn.2023-06.lalit.com:de**
```R
tgtadm -C 0 --lld iscsi --op new --mode target --tid 4 -T iqn.2023-06.lalit.com:de
tgtadm -C 0 --lld iscsi --op new --mode logicalunit --tid 4 --lun 1 -b "/dev/xvde"
# Write cache is enabled (default) for lun 1.
tgtadm -C 0 --lld iscsi --op bind --mode target --tid 4 -I 172.31.4.196
```

:o: **iSCSI Initiator**
```R
# apt-get install open-iscsi -y
```
:arrow_right: **discover the iSCSI target server to find out the shared targets.**
```R
root@iscsi-initiator:/mnt# iscsiadm --mode discovery --type sendtargets --portal  172.31.14.126:3260
root@iscsi-initiator:/mnt# iscsiadm -m discovery -t st -p 172.31.14.126
root@iscsi-initiator:/mnt# iscsiadm --mode discoverydb --type sendtargets --portal 172.31.14.126 --discover
```

:arrow_right: **Configuration example**

```R
root@iscsi-initiator:/etc/iscsi# cat initiatorname.iscsi
InitiatorName=iqn.2023-06.lalit.com:db
InitiatorName=iqn.2023-06.lalit.com:dc
InitiatorName=iqn.2023-06.lalit.com:dd
InitiatorName=iqn.2023-06.lalit.com:de
root@iscsi-initiator:/etc/iscsi#
```

:arrow_right: **Restart the iSCSI initiator service.**

```R
systemctl restart open-iscsi iscsid
systemctl status open-iscsi
```

:arrow_right: **Login to target to see Virtual disk from target.**

```R
iscsiadm -m node --login
iscsiadm --mode node --targetname iqn.2023-06.lalit.com:db --portal 172.31.14.126:3260 --login
```

:arrow_right: **Session**

```R
iscsiadm -m session

iscsiadm -m session -o show

```
:arror_right: **Check virtual disk.**

```R
Disk /dev/sda: 1 GiB, 1073741824 bytes, 2097152 sectors
Disk model: VIRTUAL-DISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```

:arrow_right: **Detail of specific target.**
```R
iscsiadm --mode node --targetname iqn.2023-06.lalit.com:db --portal 172.31.14.126:3260
```

:arrow_right: **Logout.**
```R
iscsiadm -m node --logout
```

## Troubleshooting

:red_circle: **Bioturing ISCSI - Not showing Virtual DISK**

:star2: *When we create targets is created Logical disk ( LUN1).* 
:star2: **Issue:** Virtual disk not able see even session was OK.
:star2: **Resolution:** Add logical drive to LUN

```R
tgtadm --lld iscsi --op new --mode logicalunit --tid 4 --lun 1 -b /dev/md3
```

:arrow_forward: **Restart Services**
```R
sudo systemctl restart iscsid.service

systemctl restart tgt

```

:large_blue_diamond: **Timezone:** very important

```R
timedatectl set-timezone Etc/UTC
timedatectl set-local-rtc 0
timedatectl
```

```R
============================
```

:red_circle: **Mount with No UUID**

```R
xfs_admin -U generate /dev/sdxxx

mount -t xfs -o nouuid /dev/raidxxxx /mnt/mdxxxx

mount -t xfs -o nouuid /dev/sdi /mnt/ssd

```

```R
============================
```
