### Extending a logical volume in ESXI

#Information:

For the sake of this documentation the new disk will in all commands be assumed to be sdb. This can vary in reality.

#Make the machine recognize the new disk
- SSH on the VM, become root
- Trigger machine to see new disk:
- Debian / Ubuntu:
```sh
rescan-scsi-bus
```

Example Output:
```sh
/sbin/rescan-scsi-bus: line 592: [: 1.43: integer expression expected
Host adapter 0 (ata_piix) found.
Host adapter 1 (ata_piix) found.
Host adapter 2 (mptspi) found.
Scanning SCSI subsystem for new devices
Scanning host 0 for  SCSI target IDs  0 1 2 3 4 5 6 7, all LUNs
Scanning host 1 for  SCSI target IDs  0 1 2 3 4 5 6 7, all LUNs
Scanning host 2 for  SCSI target IDs  0 1 2 3 4 5 6 7, all LUNs
Scanning for device 2 0 0 0 ...
OLD: Host: scsi2 Channel: 00 Id: 00 Lun: 00
      Vendor: VMware   Model: Virtual disk     Rev: 2.0
      Type:   Direct-Access                    ANSI SCSI revision: 06
Scanning for device 2 0 1 0 ...
OLD: Host: scsi2 Channel: 00 Id: 01 Lun: 00
      Vendor: VMware   Model: Virtual disk     Rev: 2.0
      Type:   Direct-Access                    ANSI SCSI revision: 06
Scanning for device 2 0 2 0 ...
NEW: Host: scsi2 Channel: 00 Id: 02 Lun: 00
      Vendor: VMware   Model: Virtual disk     Rev: 2.0
      Type:   Direct-Access                    ANSI SCSI revision: 06
1 new device(s) found.
0 device(s) removed.
```

If you get response that the command "rescan-scsi-bus" was not found execute this command:
```sh
ls /sys/class/scsi_host/ | while read host ; do echo "- - -" > /sys/class/scsi_host/$host/scan ; done
```

- CentOS:
```sh
echo "- - -" > /sys/class/scsi_host/host0/scan
echo "- - -" > /sys/class/scsi_host/host1/scan
echo "- - -" > /sys/class/scsi_host/host2/scan
```

To see that the new disk was recognized and what the name is you can execute:
- lsblk (new device is listed at the bottom (e.g. sdb)):
```sh
NAME                MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
fd0                   2:0    1     4K  0 disk
sda                   8:0    0   200G  0 disk
|-sda1                8:1    0     1M  0 part
|-sda2                8:2    0   477M  0 part /boot
`-sda3                8:3    0 199.5G  0 part
  |-vg-root         252:0    0   9.3G  0 lvm  /
  |-vg-swap_1       252:1    0   976M  0 lvm  [SWAP]
  |-vg-var+log      252:2    0   4.7G  0 lvm  /var/log
  |-vg-mysql_data   252:3    0   150G  0 lvm  /var/lib/mysql
  `-vg-mysql_binlog 252:4    0  34.6G  0 lvm  /var/lib/mysql_binlog
sdb                   8:16   0    50G  0 disk
```

- dmesg (information about new device is printed at the end (e.g. sdb)):
```sh
[1214009.574000] scsi 0:0:1:0: Direct-Access     VMware   Virtual disk     1.0  PQ: 0 ANSI: 2
[1214009.574012] scsi target0:0:1: Beginning Domain Validation
[1214009.574661] scsi target0:0:1: Domain Validation skipping write tests
[1214009.574664] scsi target0:0:1: Ending Domain Validation
[1214009.574714] scsi target0:0:1: FAST-40 WIDE SCSI 80.0 MB/s ST (25 ns, offset 127)
[1214009.576143] sd 0:0:1:0: [sdb] 83886080 512-byte logical blocks: (42.9 GB/40.0 GiB)
[1214009.576167] sd 0:0:1:0: [sdb] Write Protect is off
[1214009.576169] sd 0:0:1:0: [sdb] Mode Sense: 61 00 00 00
[1214009.576215] sd 0:0:1:0: [sdb] Cache data unavailable
[1214009.576217] sd 0:0:1:0: [sdb] Assuming drive cache: write through
[1214009.576475] sd 0:0:1:0: [sdb] Cache data unavailable
[1214009.576477] sd 0:0:1:0: [sdb] Assuming drive cache: write through
[1214009.577506]  sdb: unknown partition table
[1214009.577665] sd 0:0:1:0: [sdb] Cache data unavailable
[1214009.577667] sd 0:0:1:0: [sdb] Assuming drive cache: write through
[1214009.577738] sd 0:0:1:0: [sdb] Attached SCSI disk
```

# Create physical volume on the new disk
In order to get the new disk usable in the LVM we first need to create a physical volume on it:

```sh
pvcreate /dev/sdb
```

# Add physical volume to the volume group

Next we need to add the newly created physical volume to an existing volume group. For this we first need to find the volume group:
```sh
vgs
```

This will give you an output similar to this:
```sh
VG #PV #LV #SN Attr   VSize  VFree
vg   2   7   0 wz--n- <2.20t    0
```

In this case our volume group is named vg and for the sake of this documentation we will refer to the volume group as vg from now on.

The following command will add the new physical volume to the volume group:
```sh
vgextend vg /dev/sdb
```

# Extend the logical volume
Usually you come to this document because you want to extend a certain partition on a machine. If you look at the df output of a machine you find partitions like this:
```sh
*snip*
/dev/mapper/vg-varopt                   486G  1.7G  459G   1% /var/opt
/dev/mapper/vg-gitlab_repo               98G   33G   61G  35% /var/opt/gitlab/git-data
/dev/mapper/vg-gitlab_lfs                49G  1.1G   46G   3% /var/opt/gitlab/git-lfs-data
/dev/mapper/vg-gitlab_registry          1.5T   77M  1.4T   1% /var/opt/gitlab/git-registry-data
*snip*
```

If we now look at the existing logical volumes:
```sh
lvs
```

We see a list of logical volumes with information on which volume group they are and what size they currently have.

Example output:
```sh
LV              VG Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
gitlab_lfs      vg -wi-ao----  50.00g
gitlab_registry vg -wi-ao----   1.46t
gitlab_repo     vg -wi-ao---- 100.00g
varopt          vg -wi-ao---- 493.91g
```

As we can see the logical volume names are translated to a path by prefixing them with /dev/mapper/<volume_group_name>-<logical_volume_name>.

For the sake of this documentation we use the logical volume varopt which makes the path /dev/mapper/vg-varopt.

Now we can extend the logical volume and for this we have two possibilities:
-Add all the space (100%) from the new disk
```sh
lvextend -l +100%FREE /dev/mapper/vg-varopt
```
- Only add some amount and leave some of it free as a buffer
If we for example added a 10 GB disk to the machine but we only want to add 3GB of disk space to the partition then we can use this command:
```sh
lvextend -L+3G /dev/mapper/vg-varopt
```

# Extend the filesystem
Now that we have extended the logical volume, the last step to do is to adjust the filesystem on the logical volume. For this we need the following command:

- If the filesystem is ext4:
```sh
resize2fs -p /dev/mapper/vg-varopt
```
- if the filesystems is xfs:
```sh
xfs_growfs /dev/mapper/vg-varopt
```
