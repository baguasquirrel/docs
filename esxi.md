To find an [ESXi's mac address](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=1031111):

```
esxcfg-nics -l
esxcfg-vswitch -l
esxcfg-vmknic -l
```

SSD Performance on [Skull Canyon
ESXi](http://www.virtuallyghetto.com/2016/05/heads-up-esxi-not-working-on-the-new-intel-nuc-skull-canyon.html)
is really lame:

Total disk usage (in MB/s) (over the last hour) (of which bonnie++ was running
for 35 minutes):

- Avg: 1.71
- Max: 7.91
- Latest: 2.04

Remember to always do this:

```
export TERM=xterm
```

Using [esxtop](https://recommender.vmware.com/solution/vwLlbd53Wj) to troubleshoot:

```
 4:41:21pm up 6 days 19:12, 483 worlds, 3 VMs, 5 vCPUs; CPU load average: 0.01, 0.01, 0.01

 ADAPTR PATH                 NPTH AQLEN   CMDS/s  READS/s WRITES/s MBREAD/s MBWRTN/s DAVG/cmd KAVG/cmd GAVG/cmd QAVG/cmd DAVG/rd KAVG/rd GAVG/rd QAVG/rd DAVG/wr KAVG/wr GAVG/w
 vmhba0 -                       1    31    11.30     0.99     7.93     0.00     1.92   300.66  1796.42  2097.08     0.05    0.15    0.02    0.17    0.00  313.38   90.25  403.6
vmhba32 -                       1     1     0.00     0.00     0.00     0.00     0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.0
```

Snapshot taken while `bonnie++` was running on VM.


```
[root@esxi:/var/log] df -h
Filesystem   Size   Used Available Use% Mounted on
VMFS-6     978.0G  36.1G    941.9G   4% /vmfs/volumes/ssd
vfat       249.7M   8.0K    249.7M   0% /vmfs/volumes/3071f4ed-73fd8718-f003-4b4f069e050b
vfat       285.8M 205.8M     80.0M  72% /vmfs/volumes/58688336-3e239f00-8dbc-001fc69c1ddf
vfat       249.7M 143.6M    106.1M  58% /vmfs/volumes/cdb18eec-f0051e03-18a7-8bc932eca2d5
```

Let's try setting the ScratchC

* ScratchConfig.ConfiguredScratchLocation: `/vmfs/volumes/ssd/.locker`
* ScratchConfig.CurrentScratchLocation:  `/vmfs/volumes/58689744-b4b1ddd8-519b-001fc69c1ddf/.locker`

But rebooting doesn't change the scratch location.

```
vi /etc/vmware/locker.conf
  /vmfs/volumes/ssd/.locker 0
```

I just spent 30 minutes chasing my tail: there are TWO volumes which start
with `/vmfs/volumes/5868...`, but only the USB shows up in `df` (the SSD
shows up as `/vmfs/volumes/ssd`

I see the following troubling messages in `/var/log/vmkernel.log`:

```
2017-02-25T17:34:15.505Z cpu0:65560)NMP: nmp_ThrottleLogForDevice:3617: Cmd 0x2a (0x439500a7f200, 65693) to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" on path "vmhba0:C0:T0:L0" Failed: H:0x2 D:0x0 P:0x0 Invalid sense $
2017-02-25T17:34:15.505Z cpu0:65560)WARNING: NMP: nmp_DeviceRequestFastDeviceProbe:237: NMP device "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" state in doubt; requested fast path state update...
2017-02-25T17:34:15.505Z cpu0:65560)ScsiDeviceIO: 2927: Cmd(0x439500a7f200) 0x2a, CmdSN 0xaa9 from world 65693 to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" failed H:0x2 D:0x0 P:0x0 Invalid sense data: 0x75 0x70 0x64.
2017-02-25T17:34:15.762Z cpu3:65563)NMP: nmp_ThrottleLogForDevice:3546: last error status from device t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00 repeated 10 times
```

Let's try [updating](https://miketabor.com/easy-esxi-6-0-upgrade-via-command-line/):

```
esxcli software profile update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml -p ESXi-6.5.0-20170104001-standard
```

Update did *not* fix the problem:

```
2017-02-25T17:55:58.931Z cpu2:65562)NMP: nmp_ThrottleLogForDevice:3617: Cmd 0x2a (0x439500b9e200, 66300) to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" on path "vmhba0:C0:T0:L0" Failed: H:0x2 D:0x0 P:0x0 Invalid sense $
2017-02-25T17:55:58.931Z cpu2:65562)WARNING: NMP: nmp_DeviceRequestFastDeviceProbe:237: NMP device "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" state in doubt; requested fast path state update...
2017-02-25T17:55:58.931Z cpu2:65562)ScsiDeviceIO: 2927: Cmd(0x439500b9e200) 0x2a, CmdSN 0x75d from world 66300 to dev "t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00" failed H:0x2 D:0x0 P:0x0 Invalid sense data: 0x75 0x70 0x64.
2017-02-25T17:55:59.445Z cpu6:67937)vmw_ahci[00000017]: scsiUnmapCommand:Unmap transfer 0x18 byte
2017-02-25T17:55:59.452Z cpu3:65563)NMP: nmp_ThrottleLogForDevice:3546: last error status from device t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00 repeated 10 times
```

Saw the following while running `bonnie++`:

```
2017-02-25T18:19:29.773Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.773Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.775Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.775Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.777Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.777Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.779Z cpu6:65581)vmw_ahci[00000017]: scsiTaskMgmtCommand:VMK Task: VIRT_RESET initiator=0x43029e371440
2017-02-25T18:19:29.779Z cpu6:65581)vmw_ahci[00000017]: ahciAbortIO:(curr) HWQD: 0 BusyL: 0
2017-02-25T18:19:29.783Z cpu0:68447)WARNING: FS3J: 3034: Aborting txn callerID: 0xc1d00006 to slot 211:  File system timeout (Ok to retry)
2017-02-25T18:19:29.783Z cpu3:68444)HBX: 2956: 'ssd': HB at offset 3735552 - Waiting for timed out HB:
```

In the Web Host -> Monitor -> Events, I see log messages such as "Lost access
to volume 58689...." and "Successfully restored access to volume 58689..."

Let's try resetting BIOS (ugh).

Bumped the BIOS from 0037 to 0042, and reset to defaults, and booted. Ran
`bonnie++`, saw "Lost access to volume..."

```
esxcli storage core adapter list
esxcli storage core device list
```

Smart Status:
```
esxcli storage core device smart get -d t10.ATA_____Crucial_CT1050MX300SSD4_________________________163713E82C00 | grep -v "N/A"
  Parameter                     Value  Threshold  Worst
  ----------------------------  -----  ---------  -----
  Read Error Count              100    0          100
  Power-on Hours                100    0          100
  Power Cycle Count             100    0          100
  Reallocated Sector Count      100    10         100
  Raw Read Error Rate           100    0          100
  Drive Temperature             66     0          50
  Initial Bad Block Count       100    0          100
```

Let's update the firmware on the Crucial SSD:

```
hdiutil convert -format UDRW -o /tmp/crucial.dmg ~/Downloads/mx300_revM0CR040_bootable_media_update.iso
sudo dd if=/tmp/crucial.dmg of=/dev/rdisk1 bs=1m
```

Updating: Download <https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/esx/vmw/vmw-ESXi-6.5.0-metadata.zip>
and check `/profiles/`, then run the following:

```
esxcli system version getesxcli system version get
esxcli software profile update -d https://hostupdate.vmware.com/software/VUM/PRODUCTION/main/vmw-depot-index.xml -p ESXi-6.5.0-20170404001-standard
```

When installing vcenter:
* IPv4
  * 10.0.9.105 / 24
  * 10.0.9.1
* IPv6
  * 2601:646:102:95::105/64
  * fe80::82ea:96ff:fee7:5524
