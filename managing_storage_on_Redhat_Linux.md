# Using RHEL commands to manage storage used by Oracle ASM

Once upon a time, Oracle databases and Red Hat Enterprise Linux were a thing -- it was what you did for high performance Oracle database performance.  Oracle provided RPM packages, like asmlib, to augment how Oracle would manage raw devices often attached to servers.

In 2011, despite recommendations from Red Hat, Oracle and VMware, I, along with peers at EMC (and much later VMware's Application Services team) started an "Oracle-as-a-VM" effort, using only VMDKs dedicated to FRA, DATA and REDO logs; and multiple paravirtualization SCSI controllers (pvSCSI), etc.

Unfortunately as we switched to RHEL6, and later RHEL7, Oracle dropped the availability of the asmlib package to all Red Hat Linux users. If you wanted this feature, Oracle wanted you to change (a) RHEL to Oracle Enterprise Linux (including your support contract), and (b) only rely on OVM AKA Oracle Virtual Machine.

After some initial benchmarks failed using the OLE/OVM combination, this was impractical, and we elected to spread disks for FRA (4 VMDKs), DATA (groups of 4 VMDKs if multiple data areas) and REDO (2 VMDKs) since storage at that time was still on spinning 15K RPM disks.

## Multi VMDK vs Raw Disks

Using spinning disks on a SAN, most physical servers relied on giving 4 disks to Oracle ASM for every active Oracle ASM "Disk Group" and let Oracle ASM (Grid) figure out how to manage data allocation within the disk members.

We mapped to VMDKs just so we had use of Storage VMotion feature of vSphere / vCenter Server. 

During a 13 year period, managing 4 to 10 Oracle on RHEL Linux virtual servers, we expanded the storage over 20 times and NEVER had to perform an export / import process -- 
instead adding new storage, using the storage SAN COPY or Storage VMotion (preferred) to new larger disks, and other than Linux commands and lining up the SCSI IDs in the udev file, we never had to ask a DBA to export or import a database to make use of the new storage. 

Note;  We dropped considerations of Oracle Enterprise Linux, as they dropped pvSCSI kernel support (losing their 100% compatibility with Red Hat Enterprise Linux) since it costs them benchmarks since they moved that functionality (or so they thought) to the Oracle virtualization layer (OVM).  Again, since a key operational requirement was to do seamless storage disk growth, using Storage VMotion and Linux configuration work only, we never saw OLE as viable in a VM setting.



## pvSCSI Challenge

The Linux kernel guarantees the order of the SCSI device ids on an specific SCSI controller, but never guarantees the order multiple SCSI controllers will be presented.

So initial testing was the wild west of disk usage.  Something the four controllers came up in order 

DESIRED:
  1. C0: /dev/sda to /dev/sdc (RHEL root, swap and where Oracle binaries (/u01/app) would reside
  2. C1: /dev/sdd to /dev/sdg (DATA, mapped to 4 VMDKs)
  3. C2: /dev/sdh to /dev/sdk (FRA, mapped to 4 VMDKs)
  4. C3: /dev/sdl to /dev/sdm (REDO, mapped to 2 VMDKs)

WHAT WE GOT:
  1. First boot:  C0, C1, C2, C3 - handled off to DBA for installation
  2. Next boot:  C0, C1, C2, C3 - so we didn't notice this issue, and all benchmarks went flawlessly
     ....
  4. Eighth boot: C2, C1, C0, C3 - didn't boot
  5. Ninth boot:  C0, C3, C2, C1 - fortunately database never started since Oracle ASM {Grid} wasn't happy

     This is where conversation with Red Hat and other research revealed that Order of SCSI Controllers is never guaranteed

     For Years, since ASMLIB in RHEL 5 and earlier masquerading this behavior, it was not document in any source on internet (2011)

     Had to stop and devise UDEV as ASMLIB alternative (see below)
     
  7. Every boot since:  C0, C1, C2, C3 - ran production in both RHEL6 and RHEL7 for over 10 years with never an issue with "Oracle-as-a-VM" design



# Enter UDEV and the ASMLIB Alternative

Fortunately, we had an alternative to Oracle ASMLIB and using RAW DISKS.  The "udev" capability allowed us to make VMDKs by forceing the underlying VM to display SCSI IDs for its disks (now thought of as UUIDs), so we could map SAN LUNs to ESXi Controller Storage devices to VM Datastores (AKA VMDKs) to RHEL kernel "scsi devices" and using 'udev', we could made the intended Oracle VMDKs to aptly named /dev/asm-....-devices that could be formatted and then presented to Oracle ASM (Grid) for using by Oracle

I can provide a link to the research and changes as udev specifications, and scsi_id command, changed over time, but here is a short review of the commands assuming your VM was already presenting SCSI ID to the RHEL kernel:

    ```[TO DO](provide link to the PowerCLI script to that gets VM's Advanced Settings for disk ids, and set flag to expose SCSI IDs if not present)```

##  Novel bash one-liner to confirm assignments of /dev/sdXX devices to SCSI IDs

```
[my-server:larryt:~]$ for dsk in `ls -l /dev/sd[a-z]* | awk ' {print $10 } '`; do sid=`/usr/lib/udev/scsi_id -gud $dsk`; size=`fdisk -l $dsk | grep 'Disk /dev/sd' | awk ' { print $3 }'`; echo $dsk $sid $size; done
/dev/sda 36000c299e2128679a99c5b179629222a 107.4
/dev/sda1 36000c299e2128679a99c5b179629222a 1073
/dev/sda2 36000c299e2128679a99c5b179629222a 106.3
/dev/sdb 36000c297c3385b24b27bca8eb58ceb58 68.7
/dev/sdb1 36000c297c3385b24b27bca8eb58ceb58 68.7
/dev/sdc 36000c297279037fed1f6d9b2c067fc92 128.8
/dev/sdc1 36000c297279037fed1f6d9b2c067fc92 128.8
/dev/sdd 36000c29b58136cf693eb2f36bb7fcc6b 289.9
/dev/sde 36000c293af59911daeb650174c6cf29d 289.9
/dev/sdf 36000c2991f67f948a733e5a6675b66ab 644.2
/dev/sdf1 36000c2991f67f948a733e5a6675b66ab 644.2
/dev/sdg 36000c29343f387359b53028b5e513964 343.6
/dev/sdg1 36000c29343f387359b53028b5e513964 343.6
/dev/sdh 36000c29e64f708f5b617f80569ad7522 343.6
/dev/sdh1 36000c29e64f708f5b617f80569ad7522 343.6
/dev/sdi 36000c2954d99e1bf1c7a2b67bb995a51 42.9
/dev/sdi1 36000c2954d99e1bf1c7a2b67bb995a51 42.9
/dev/sdj 36000c29e27c1793c575389737b13cf90 42.9
/dev/sdj1 36000c29e27c1793c575389737b13cf90 42.9
[my-server:larryt:~]$ 

```

## lsscsi to map current kernel boot order assigned to discovered SCSI controllers at ~this boot time~: 

```
[my-server:larryt:~]$ lsscsi
[0:0:0:0]    disk    VMware   Virtual disk     1.0   /dev/sda
[0:0:1:0]    disk    VMware   Virtual disk     1.0   /dev/sdb
[0:0:2:0]    disk    VMware   Virtual disk     1.0   /dev/sdc
[1:0:0:0]    disk    VMware   Virtual disk     1.0   /dev/sdd
[1:0:1:0]    disk    VMware   Virtual disk     1.0   /dev/sde
[2:0:0:0]    disk    VMware   Virtual disk     1.0   /dev/sdf
[2:0:1:0]    disk    VMware   Virtual disk     1.0   /dev/sdg
[2:0:2:0]    disk    VMware   Virtual disk     1.0   /dev/sdh
[3:0:0:0]    disk    VMware   Virtual disk     1.0   /dev/sdi
[3:0:1:0]    disk    VMware   Virtual disk     1.0   /dev/sdj
[6:0:0:0]    cd/dvd  NECVMWar VMware SATA CD00 1.00  /dev/sr0
[my-server:larryt:~]$

```

## Confirming mapping of DBA friendly device names to those devices assigned to Oracle ASM Disk groups

   NOTE:  If you haven't noticed, after we went to all SSD Flash Storage Array, we no longer had to present a storage to ASM in groups of 4 disk members
```
[my-server:larryt:~]$ ls -l /dev/asm*
lrwxrwxrwx. 1 root root       3 Sep 18 16:52 /dev/asm-ORA_SE2_DATA-OM-P-1 -> sdd
lrwxrwxrwx. 1 root root       3 Sep 18 16:52 /dev/asm-ORA_SE2_DATA-OM-P-2 -> sde
lrwxrwxrwx. 1 root root       4 Sep 18 16:52 /dev/asm-ORA_SE2_FRA-OM-P-1 -> sdg1
lrwxrwxrwx. 1 root root       4 Sep 18 16:31 /dev/asm-ORA_SE2_FRA-OM-P-2 -> sdh1
lrwxrwxrwx. 1 root root       4 Sep 18 16:52 /dev/asm-ORA_SE2_REDO-OM-P-1 -> sdi1
lrwxrwxrwx. 1 root root       4 Sep 18 16:31 /dev/asm-ORA_SE2_REDO-OM-P-2 -> sdj1

/dev/asm:
total 0
[my-server:larryt:~]$ 
```

# UDEV configuration on RHEL7

```
my-server:larryt:~]$  ls -l /etc/udev/rules.d/
total 12
-rw-r--r--. 1 root root  224 May 11 18:56 53-afd.rules
-rw-r--r--. 1 root root  353 May 11 18:57 55-usm.rules
-rw-r--r--. 1 root root 1042 Mar 14  2019 99-oracle-asmdevices.rules
my-server:larryt:~]$ 
```

##TO DO   

 - Add section on 99-oracle-asmdevices.rules configuration
 - If requested, give summary of OLE/OVM vs UDEV/RHEL/VMware benchmark wall times, etc.
   
