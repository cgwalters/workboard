# OSTree update niceness vs etcd

https://github.com/openshift/machine-config-operator/issues/1897

This is certainly a fantastically complex topic.  I have also seen desktop OSTree users complain about interactivity during updates.

A few further thoughts:

### Control plane vs workers

We want to avoid disrupting etcd (and the control plane in general), but: while we also want to avoid disrupting *workers*, I think we shouldn't let user workloads completely block OS updates either.

### oscontainer wrapping forcing large I/O

The way we pull the whole oscontainer and unpack it means we are always doing 1G+ of I/O, even for small OS updates.  There are several entirely different ways to fix that:

#### Run a HTTP server (kube service) that mounts the oscontainer

This would have several huge advantages, among them that OS updates suddenly work exactly how OSTree was designed from the start - we only write *new* objects to disk.  It would also avoid SELinux problems we've hit trying to get content from the oscontainer to the host.  But it'd be a nontrivial change that would conflict with ongoing work on extensions.

We'd need to be careful to not mount the oscontainer on workers and then pull to the control plane, or we escalate potential worker compromise to control plane.  (Or we need to e.g. GPG sign the oscontainer content)

#### Stream unpacking the oscontainer

Rather than use `podman pull`+`podman mount`, stream the oscontainer and unpack it as we go.  I think the main disadvantage of this is that it'd be a new nontrivial bit of code, and we'd need to do some tricky things like still verify the compressed sha256 at the end, and discard all intermediate work if that fails to verify.

# The IO scheduler

In current Fedora but not yet RHEL8, [bfq is the default](https://github.com/systemd/systemd/pull/13321).  Currently RHEL8 defaults to `mq-deadline`.   And one thing I notice here is that if I do `ionice -c 3 ostree pull ...` it's about a 20% boost for the non-nice test `dd` workload but only if `bfq` is enabled.

# Test criteria

```
#!/bin/bash
# Setup script - suitable to run in a container image, but
# we expect $(pwd) to be a local XFS bind mounted in
# and not the container's overlayfs
set -xeuo pipefail
rm repo -rf
ostree --repo=repo init --mode=bare-user
img42=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:b64e472b57538ebd6808a1e0528d9ea83877207d29c091634f41627a609f9b04
commit42=f2126cdca6a924938072543aa9d6df4436fe72462f59d3d2ad97794458cb5550
rm tmp -rf
mkdir tmp
oc image extract "${img42}" --path=/srv/repo:tmp
ostree --repo=repo pull-local tmp/repo "${commit42}"
rm tmp -rf
# Flush to avoid mixing in the prep pull with later tests
sync -f .
```

```
#!/bin/bash
# Like above, expects . to be the workdir
# You likely want to mirror this locally, but we pull it because
# we want to simulate all of the temporary data that gets written
# by the podman stack as well.
set -xeuo pipefail
ostree --repo=repo prune --refs-only --depth=0
img45=quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:784456823004f41b2718e45796423bd8bf6689ed6372b66f2e2f4db8e7d6bcb9
commit45=bcfae65a2ab0b4ab5afcd1c9f1cce30b300e5408b04fc8bdec51a143ab593d40
rm tmp etcdtest -rf
sync -f .
# Now, begin the test
mkdir etcdtest tmp
rm -f fio.log
fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd > fio.log &
oc image extract "${img45}" --path=/srv/repo:tmp
ostree --repo=repo pull-local tmp/repo ${commit45}
wait
cat fio.log
```

# Results

Baseline test setup: 

- Bare metal i9-9900k workstation
- kernel: 5.7.8-200.fc32.x86_64
- filesystem: xfs
- block device: Samsung SSD 970 EVO 1TB (NVMe), 512G partition
- scheduler: `[none]`
 
### Baseline fio

```
[root@toolbox testetcd]# fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][w=1970KiB/s][w=877 IOPS][eta 00m:00s]
etcd: (groupid=0, jobs=1): err= 0: pid=408906: Wed Jul 15 02:14:39 2020
  write: IOPS=858, BW=1928KiB/s (1974kB/s)(29.0MiB/15937msec); 0 zone resets
    clat (nsec): min=1024, max=14349k, avg=103520.77, stdev=145454.55
     lat (nsec): min=1082, max=14350k, avg=104346.19, stdev=145488.30
    clat percentiles (usec):
     |  1.00th=[    5],  5.00th=[    7], 10.00th=[   14], 20.00th=[   25],
     | 30.00th=[   25], 40.00th=[   26], 50.00th=[  108], 60.00th=[  147],
     | 70.00th=[  180], 80.00th=[  188], 90.00th=[  200], 95.00th=[  206],
     | 99.00th=[  217], 99.50th=[  225], 99.90th=[  545], 99.95th=[  553],
     | 99.99th=[  586]
   bw (  KiB/s): min= 1832, max= 2061, per=100.00%, avg=1933.16, stdev=56.02, samples=31
   iops        : min=  816, max=  918, avg=860.87, stdev=24.98, samples=31
  lat (usec)   : 2=0.10%, 4=0.78%, 10=5.57%, 20=6.89%, 50=30.48%
  lat (usec)   : 100=3.96%, 250=51.96%, 500=0.16%, 750=0.10%
  lat (msec)   : 20=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=732, max=4286, avg=1052.51, stdev=278.42
    sync percentiles (usec):
     |  1.00th=[  848],  5.00th=[  906], 10.00th=[  930], 20.00th=[  963],
     | 30.00th=[  988], 40.00th=[ 1020], 50.00th=[ 1037], 60.00th=[ 1045],
     | 70.00th=[ 1074], 80.00th=[ 1090], 90.00th=[ 1123], 95.00th=[ 1123],
     | 99.00th=[ 1434], 99.50th=[ 3884], 99.90th=[ 4113], 99.95th=[ 4178],
     | 99.99th=[ 4228]
  cpu          : usr=1.14%, sys=5.69%, ctx=48686, majf=0, minf=16
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=1928KiB/s (1974kB/s), 1928KiB/s-1928KiB/s (1974kB/s-1974kB/s), io=29.0MiB (31.5MB), run=15937-15937msec

Disk stats (read/write):
  nvme1n1: ios=7674/27335, merge=0/0, ticks=909/13768, in_queue=27707, util=99.52%
```

### With concurrent update

```
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=408818: Wed Jul 15 02:12:16 2020
  write: IOPS=381, BW=858KiB/s (878kB/s)(29.0MiB/35815msec); 0 zone resets
    clat (usec): min=2, max=434, avg=18.88, stdev=12.67
     lat (usec): min=3, max=435, avg=19.46, stdev=12.96
    clat percentiles (usec):
     |  1.00th=[    5],  5.00th=[    5], 10.00th=[    6], 20.00th=[    7],
     | 30.00th=[    9], 40.00th=[   12], 50.00th=[   17], 60.00th=[   25],
     | 70.00th=[   26], 80.00th=[   32], 90.00th=[   36], 95.00th=[   37],
     | 99.00th=[   47], 99.50th=[   52], 99.90th=[   60], 99.95th=[   66],
     | 99.99th=[  343]
   bw (  KiB/s): min=    4, max= 1055, per=100.00%, avg=908.57, stdev=253.13, samples=67
   iops        : min=    2, max=  470, avg=404.72, stdev=112.68, samples=67
  lat (usec)   : 4=0.30%, 10=34.99%, 20=19.16%, 50=44.92%, 100=0.60%
  lat (usec)   : 250=0.01%, 500=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=732, max=1935.7k, avg=2594.14, stdev=19014.96
    sync percentiles (usec):
     |  1.00th=[   832],  5.00th=[   881], 10.00th=[   914], 20.00th=[   971],
     | 30.00th=[  1020], 40.00th=[  1090], 50.00th=[  2933], 60.00th=[  3032],
     | 70.00th=[  3064], 80.00th=[  3130], 90.00th=[  3228], 95.00th=[  3425],
     | 99.00th=[  6194], 99.50th=[  7963], 99.90th=[ 17433], 99.95th=[ 95945],
     | 99.99th=[843056]
  cpu          : usr=0.37%, sys=2.55%, ctx=41884, majf=0, minf=15
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=858KiB/s (878kB/s), 858KiB/s-858KiB/s (878kB/s-878kB/s), io=29.0MiB (31.5MB), run=35815-35815msec

Disk stats (read/write):
  nvme1n1: ios=0/43702, merge=0/69266, ticks=0/385455, in_queue=402971, util=99.51%
```

Note that the bandwidth is about halved, the 99th percentile is still under 10ms...but the standard deviation shoots up; there are a few *much, much* longer fsyncs, including one that took nearly a full 2 seconds.

### Attempted tweaks that didn't help

- change scheduler to `bfq`
- Use `ionice -c 3` (with and without `bfq`) EDIT: see below

### Use `--disable-fsync` for ostree pull: Big Improvement!

```
  fsync/fdatasync/sync_file_range:
    sync (usec): min=756, max=767233, avg=2449.79, stdev=7291.31
    sync percentiles (usec):
     |  1.00th=[   848],  5.00th=[   906], 10.00th=[   947], 20.00th=[   996],
     | 30.00th=[  1045], 40.00th=[  1106], 50.00th=[  2966], 60.00th=[  3064],
     | 70.00th=[  3130], 80.00th=[  3195], 90.00th=[  3261], 95.00th=[  3589],
     | 99.00th=[  5932], 99.50th=[  6456], 99.90th=[ 23462], 99.95th=[ 96994],
     | 99.99th=[200279]
```

# Reworking ostree to reduce fsync "spike"

https://github.com/ostreedev/ostree/pull/2147

```
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=523830: Wed Jul 15 03:38:32 2020
  write: IOPS=249, BW=561KiB/s (575kB/s)(29.0MiB/54720msec); 0 zone resets
    clat (usec): min=2, max=425, avg=23.27, stdev=15.17
     lat (usec): min=2, max=426, avg=24.00, stdev=15.50
    clat percentiles (usec):
     |  1.00th=[    5],  5.00th=[    5], 10.00th=[    6], 20.00th=[    7],
     | 30.00th=[   11], 40.00th=[   24], 50.00th=[   26], 60.00th=[   28],
     | 70.00th=[   34], 80.00th=[   36], 90.00th=[   38], 95.00th=[   42],
     | 99.00th=[   55], 99.50th=[   60], 99.90th=[   87], 99.95th=[  129],
     | 99.99th=[  388]
   bw (  KiB/s): min=   13, max= 1078, per=99.20%, avg=556.51, stdev=278.53, samples=108
   iops        : min=    6, max=  480, avg=248.02, stdev=124.00, samples=108
  lat (usec)   : 4=0.33%, 10=29.25%, 20=8.26%, 50=60.68%, 100=1.42%
  lat (usec)   : 250=0.03%, 500=0.04%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=741, max=461949, avg=3970.41, stdev=6696.28
    sync percentiles (usec):
     |  1.00th=[   824],  5.00th=[   898], 10.00th=[   947], 20.00th=[  1029],
     | 30.00th=[  2073], 40.00th=[  2868], 50.00th=[  3032], 60.00th=[  3163],
     | 70.00th=[  4113], 80.00th=[  6194], 90.00th=[  8586], 95.00th=[ 10028],
     | 99.00th=[ 13042], 99.50th=[ 15139], 99.90th=[ 35390], 99.95th=[ 87557],
     | 99.99th=[459277]
  cpu          : usr=0.33%, sys=2.39%, ctx=50623, majf=0, minf=15
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=561KiB/s (575kB/s), 561KiB/s-561KiB/s (575kB/s-575kB/s), io=29.0MiB (31.5MB), run=54720-54720msec

Disk stats (read/write):
  nvme1n1: ios=0/110570, merge=0/42598, ticks=0/355020, in_queue=426989, util=99.32%
```

# Testing an Azure image

```
oc get clusterversion
NAME      VERSION                             AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.6.0-0.nightly-2020-07-15-111743   True        False         88m     Cluster version is 4.6.0-0.nightly-2020-07-15-111743
```

### Baseline

```
sh-5.0# fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)
Jobs: 1 (f=1): [W(1)][100.0%][w=762KiB/s][w=339 IOPS][eta 00m:00s]
etcd: (groupid=0, jobs=1): err= 0: pid=210771: Wed Jul 15 19:49:20 2020
  write: IOPS=344, BW=774KiB/s (793kB/s)(29.0MiB/39682msec); 0 zone resets
    clat (usec): min=4, max=1813, avg=15.10, stdev=17.69
     lat (usec): min=5, max=1814, avg=16.27, stdev=17.90
    clat percentiles (usec):
     |  1.00th=[    7],  5.00th=[    8], 10.00th=[    9], 20.00th=[   11],
     | 30.00th=[   11], 40.00th=[   12], 50.00th=[   13], 60.00th=[   15],
     | 70.00th=[   16], 80.00th=[   19], 90.00th=[   24], 95.00th=[   30],
     | 99.00th=[   40], 99.50th=[   59], 99.90th=[   91], 99.95th=[  167],
     | 99.99th=[  293]
   bw (  KiB/s): min=  566, max=  893, per=100.00%, avg=774.19, stdev=71.70, samples=78
   iops        : min=  252, max=  398, avg=344.79, stdev=31.90, samples=78
  lat (usec)   : 10=17.85%, 20=65.13%, 50=16.38%, 100=0.56%, 250=0.07%
  lat (usec)   : 500=0.01%
  lat (msec)   : 2=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=1158, max=59089, avg=2877.18, stdev=2720.90
    sync percentiles (usec):
     |  1.00th=[ 1287],  5.00th=[ 1385], 10.00th=[ 1434], 20.00th=[ 1532],
     | 30.00th=[ 1631], 40.00th=[ 2180], 50.00th=[ 2933], 60.00th=[ 3097],
     | 70.00th=[ 3228], 80.00th=[ 3425], 90.00th=[ 4047], 95.00th=[ 4948],
     | 99.00th=[ 7635], 99.50th=[11731], 99.90th=[47973], 99.95th=[50070],
     | 99.99th=[52691]
  cpu          : usr=0.50%, sys=3.44%, ctx=40213, majf=0, minf=15
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=774KiB/s (793kB/s), 774KiB/s-774KiB/s (793kB/s-793kB/s), io=29.0MiB (31.5MB), run=39682-39682msec

Disk stats (read/write):
    dm-0: ios=0/29491, merge=0/0, ticks=0/74603, in_queue=74603, util=98.97%, aggrios=0/29054, aggrmerge=0/607, aggrticks=0/73353, aggrin_queue=59120, aggrutil=98.95%
  sda: ios=0/29054, merge=0/607, ticks=0/73353, in_queue=59120, util=98.95%
```

### Azure test

```
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=24183: Wed Jul 15 21:10:14 2020
  write: IOPS=153, BW=344KiB/s (352kB/s)(29.0MiB/89381msec); 0 zone resets
    clat (usec): min=5, max=2564.7k, avg=392.34, stdev=24133.65
     lat (usec): min=6, max=2564.7k, avg=393.52, stdev=24133.66
    clat percentiles (usec):
     |  1.00th=[     8],  5.00th=[     9], 10.00th=[    10], 20.00th=[    11],
     | 30.00th=[    12], 40.00th=[    13], 50.00th=[    13], 60.00th=[    14],
     | 70.00th=[    15], 80.00th=[    16], 90.00th=[    18], 95.00th=[    20],
     | 99.00th=[    38], 99.50th=[    52], 99.90th=[  2802], 99.95th=[ 67634],
     | 99.99th=[893387]
   bw (  KiB/s): min=    4, max=  893, per=100.00%, avg=587.39, stdev=333.14, samples=104
   iops        : min=    1, max=  398, avg=261.62, stdev=148.35, samples=104
  lat (usec)   : 10=15.36%, 20=80.22%, 50=3.88%, 100=0.29%, 250=0.12%
  lat (usec)   : 500=0.01%, 1000=0.01%
  lat (msec)   : 2=0.01%, 4=0.02%, 10=0.01%, 50=0.01%, 100=0.02%
  lat (msec)   : 250=0.01%, 500=0.02%, 1000=0.01%, >=2000=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=1129, max=20882k, avg=6133.45, stdev=186657.30
    sync percentiles (usec):
     |  1.00th=[   1303],  5.00th=[   1385], 10.00th=[   1450],
     | 20.00th=[   1549], 30.00th=[   1680], 40.00th=[   2040],
     | 50.00th=[   3032], 60.00th=[   3195], 70.00th=[   3359],
     | 80.00th=[   3523], 90.00th=[   4178], 95.00th=[   5342],
     | 99.00th=[  10159], 99.50th=[  20055], 99.90th=[ 541066],
     | 99.95th=[1283458], 99.99th=[3338666]
  cpu          : usr=0.29%, sys=1.33%, ctx=40369, majf=0, minf=17
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=344KiB/s (352kB/s), 344KiB/s-344KiB/s (352kB/s-352kB/s), io=29.0MiB (31.5MB), run=89381-89381msec

Disk stats (read/write):
    dm-0: ios=0/133606, merge=0/0, ticks=0/35235809, in_queue=35235809, util=90.05%, aggrios=0/62127, aggrmerge=0/72439, aggrticks=0/10366418, aggrin_queue=10334491, aggrutil=97.78%
  sdb: ios=0/62127, merge=0/72439, ticks=0/10366418, in_queue=10334491, util=97.78%
```

### With --disable-fsync

```
+ cat fio.log
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=27343: Wed Jul 15 21:14:35 2020
  write: IOPS=144, BW=324KiB/s (332kB/s)(29.0MiB/94837msec); 0 zone resets
    clat (usec): min=5, max=3011.4k, avg=524.63, stdev=32555.22
     lat (usec): min=6, max=3011.4k, avg=525.83, stdev=32555.23
    clat percentiles (usec):
     |  1.00th=[      8],  5.00th=[     10], 10.00th=[     10],
     | 20.00th=[     11], 30.00th=[     12], 40.00th=[     13],
     | 50.00th=[     14], 60.00th=[     15], 70.00th=[     16],
     | 80.00th=[     17], 90.00th=[     18], 95.00th=[     20],
     | 99.00th=[     41], 99.50th=[     63], 99.90th=[   1188],
     | 99.95th=[  40633], 99.99th=[1786774]
   bw (  KiB/s): min=    4, max=  736, per=100.00%, avg=484.92, stdev=266.57, samples=125
   iops        : min=    1, max=  328, avg=216.08, stdev=118.78, samples=125
  lat (usec)   : 10=11.54%, 20=83.52%, 50=4.26%, 100=0.35%, 250=0.13%
  lat (usec)   : 500=0.06%, 750=0.01%, 1000=0.01%
  lat (msec)   : 2=0.04%, 10=0.01%, 50=0.02%, 100=0.01%, 500=0.01%
  lat (msec)   : 2000=0.01%, >=2000=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=1227, max=6395.6k, avg=6400.02, stdev=89958.86
    sync percentiles (usec):
     |  1.00th=[   1352],  5.00th=[   1434], 10.00th=[   1500],
     | 20.00th=[   1598], 30.00th=[   1729], 40.00th=[   2933],
     | 50.00th=[   3195], 60.00th=[   3359], 70.00th=[   3556],
     | 80.00th=[   4686], 90.00th=[   6915], 95.00th=[   8455],
     | 99.00th=[  15008], 99.50th=[  39060], 99.90th=[ 700449],
     | 99.95th=[1702888], 99.99th=[5402264]
  cpu          : usr=0.27%, sys=1.35%, ctx=40186, majf=0, minf=17
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=324KiB/s (332kB/s), 324KiB/s-324KiB/s (332kB/s-332kB/s), io=29.0MiB (31.5MB), run=94837-94837msec

Disk stats (read/write):
    dm-0: ios=0/133701, merge=0/0, ticks=0/29925616, in_queue=29925616, util=94.25%, aggrios=0/65679, aggrmerge=0/68405, aggrticks=0/8942485, aggrin_queue=8908826, aggrutil=97.75%
  sdb: ios=0/65679, merge=0/68405, ticks=0/8942485, in_queue=8908826, util=97.75%
```

### With per-object fsync

```
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=38096: Wed Jul 15 21:28:59 2020
  write: IOPS=139, BW=314KiB/s (321kB/s)(29.0MiB/97886msec); 0 zone resets
    clat (usec): min=5, max=23328, avg=15.65, stdev=202.20
     lat (usec): min=6, max=23329, avg=16.89, stdev=202.21
    clat percentiles (usec):
     |  1.00th=[    8],  5.00th=[    9], 10.00th=[   10], 20.00th=[   11],
     | 30.00th=[   11], 40.00th=[   12], 50.00th=[   13], 60.00th=[   13],
     | 70.00th=[   15], 80.00th=[   16], 90.00th=[   18], 95.00th=[   21],
     | 99.00th=[   38], 99.50th=[   55], 99.90th=[  159], 99.95th=[  322],
     | 99.99th=[ 3589]
   bw (  KiB/s): min=    4, max=  687, per=100.00%, avg=391.78, stdev=214.26, samples=156
   iops        : min=    1, max=  306, avg=174.59, stdev=95.43, samples=156
  lat (usec)   : 10=18.21%, 20=76.57%, 50=4.67%, 100=0.37%, 250=0.11%
  lat (usec)   : 500=0.04%, 750=0.01%
  lat (msec)   : 2=0.01%, 4=0.01%, 50=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=1138, max=4698.7k, avg=7131.19, stdev=78272.72
    sync percentiles (usec):
     |  1.00th=[   1254],  5.00th=[   1369], 10.00th=[   1450],
     | 20.00th=[   1565], 30.00th=[   1696], 40.00th=[   3064],
     | 50.00th=[   3326], 60.00th=[   3556], 70.00th=[   4146],
     | 80.00th=[   5276], 90.00th=[   7373], 95.00th=[  10683],
     | 99.00th=[  53740], 99.50th=[  96994], 99.90th=[ 455082],
     | 99.95th=[1753220], 99.99th=[4664067]
  cpu          : usr=0.27%, sys=1.06%, ctx=43322, majf=0, minf=15
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=314KiB/s (321kB/s), 314KiB/s-314KiB/s (321kB/s-321kB/s), io=29.0MiB (31.5MB), run=97886-97886msec

Disk stats (read/write):
    dm-0: ios=0/171886, merge=0/0, ticks=0/20713352, in_queue=20713352, util=93.70%, aggrios=0/118347, aggrmerge=0/56471, aggrticks=0/6753416, aggrin_queue=6695902, aggrutil=95.64%
  sdb: ios=0/118347, merge=0/56471, ticks=0/6753416, in_queue=6695902, util=95.64%
```

# Current work

`pull: Add --fsync-incremental`: https://github.com/ostreedev/ostree/pull/2152


# More on io scheduler (ionice -c 3 and bfq)

I discovered that *in combination* with `--fsync-incremental` above, using `ionice -c 3` also has a major impact.  It does have a ~4% performance hit on this NVMe, but in return respects idle io priority.

### baseline fio with "none"

```
[root@toolbox testetcd]# fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][w=1936KiB/s][w=862 IOPS][eta 00m:00s]
etcd: (groupid=0, jobs=1): err= 0: pid=1022655: Fri Jul 17 18:31:44 2020
  write: IOPS=855, BW=1922KiB/s (1968kB/s)(29.0MiB/15984msec); 0 zone resets
    clat (nsec): min=1914, max=14474k, avg=104661.09, stdev=145525.37
     lat (nsec): min=1999, max=14476k, avg=105534.26, stdev=145559.29
    clat percentiles (usec):
     |  1.00th=[    9],  5.00th=[   13], 10.00th=[   24], 20.00th=[   25],
     | 30.00th=[   25], 40.00th=[   27], 50.00th=[  108], 60.00th=[  151],
     | 70.00th=[  176], 80.00th=[  188], 90.00th=[  200], 95.00th=[  206],
     | 99.00th=[  219], 99.50th=[  227], 99.90th=[  490], 99.95th=[  545],
     | 99.99th=[  578]
   bw (  KiB/s): min= 1744, max= 2151, per=100.00%, avg=1926.94, stdev=75.61, samples=31
   iops        : min=  776, max=  958, avg=858.06, stdev=33.75, samples=31
  lat (usec)   : 2=0.02%, 4=0.16%, 10=1.21%, 20=7.70%, 50=34.72%
  lat (usec)   : 100=3.77%, 250=52.18%, 500=0.16%, 750=0.09%
  lat (msec)   : 20=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=777, max=6901, avg=1053.77, stdev=282.98
    sync percentiles (usec):
     |  1.00th=[  848],  5.00th=[  906], 10.00th=[  938], 20.00th=[  963],
     | 30.00th=[  996], 40.00th=[ 1020], 50.00th=[ 1037], 60.00th=[ 1045],
     | 70.00th=[ 1074], 80.00th=[ 1090], 90.00th=[ 1106], 95.00th=[ 1123],
     | 99.00th=[ 1434], 99.50th=[ 3916], 99.90th=[ 4080], 99.95th=[ 4146],
     | 99.99th=[ 4178]
  cpu          : usr=1.24%, sys=6.14%, ctx=48719, majf=0, minf=16
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=1922KiB/s (1968kB/s), 1922KiB/s-1922KiB/s (1968kB/s-1968kB/s), io=29.0MiB (31.5MB), run=15984-15984msec

Disk stats (read/write):
  nvme1n1: ios=7654/27262, merge=0/0, ticks=879/13721, in_queue=27576, util=99.50%
```

### Baseline fio with bfq

```
[root@toolbox testetcd]# fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
Jobs: 1 (f=1): [W(1)][100.0%][w=1864KiB/s][w=830 IOPS][eta 00m:00s]
etcd: (groupid=0, jobs=1): err= 0: pid=1022626: Fri Jul 17 18:31:15 2020
  write: IOPS=824, BW=1851KiB/s (1896kB/s)(29.0MiB/16593msec); 0 zone resets
    clat (usec): min=2, max=14420, avg=111.49, stdev=148.27
     lat (usec): min=2, max=14421, avg=112.42, stdev=148.29
    clat percentiles (usec):
     |  1.00th=[    7],  5.00th=[   16], 10.00th=[   20], 20.00th=[   25],
     | 30.00th=[   26], 40.00th=[   28], 50.00th=[  128], 60.00th=[  153],
     | 70.00th=[  172], 80.00th=[  204], 90.00th=[  221], 95.00th=[  229],
     | 99.00th=[  245], 99.50th=[  253], 99.90th=[  490], 99.95th=[  545],
     | 99.99th=[  586]
   bw (  KiB/s): min= 1774, max= 1994, per=100.00%, avg=1852.56, stdev=47.75, samples=32
   iops        : min=  790, max=  888, avg=824.97, stdev=21.28, samples=32
  lat (usec)   : 4=0.23%, 10=1.46%, 20=8.94%, 50=33.14%, 100=1.12%
  lat (usec)   : 250=54.48%, 500=0.55%, 750=0.08%
  lat (msec)   : 20=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=783, max=6958, avg=1091.74, stdev=255.37
    sync percentiles (usec):
     |  1.00th=[  889],  5.00th=[  947], 10.00th=[  979], 20.00th=[ 1012],
     | 30.00th=[ 1037], 40.00th=[ 1057], 50.00th=[ 1074], 60.00th=[ 1090],
     | 70.00th=[ 1106], 80.00th=[ 1139], 90.00th=[ 1156], 95.00th=[ 1172],
     | 99.00th=[ 1450], 99.50th=[ 3884], 99.90th=[ 4146], 99.95th=[ 4228],
     | 99.99th=[ 4424]
  cpu          : usr=0.97%, sys=6.13%, ctx=56322, majf=0, minf=16
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=1851KiB/s (1896kB/s), 1851KiB/s-1851KiB/s (1896kB/s-1896kB/s), io=29.0MiB (31.5MB), run=16593-16593msec

Disk stats (read/write):
  nvme1n1: ios=7601/27072, merge=0/0, ticks=960/13960, in_queue=27951, util=99.55%
```


# Concurrent updates with "none" scheduler

```
+ chrt --idle 0 ionice -c 3 ostree --repo=repo pull-local --fsync-incremental srv-repo/repo bcfae65a2ab0b4ab5afcd1c9f1cce30b300e5408b04fc8bdec51a143ab593d40
+ fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd
7602 metadata, 30681 content objects imported                                                                                                                                                                                                                   

real	0m31.721s
user	0m24.530s
sys	0m17.542s
+ wait
+ cat fio.log
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=1023173: Fri Jul 17 18:41:16 2020
  write: IOPS=278, BW=626KiB/s (641kB/s)(29.0MiB/49052msec); 0 zone resets
    clat (usec): min=2, max=381, avg=21.96, stdev=14.81
     lat (usec): min=3, max=408, avg=22.70, stdev=15.80
    clat percentiles (usec):
     |  1.00th=[    5],  5.00th=[    5], 10.00th=[    5], 20.00th=[    7],
     | 30.00th=[   10], 40.00th=[   17], 50.00th=[   25], 60.00th=[   27],
     | 70.00th=[   30], 80.00th=[   35], 90.00th=[   38], 95.00th=[   42],
     | 99.00th=[   59], 99.50th=[   72], 99.90th=[  110], 99.95th=[  149],
     | 99.99th=[  359]
   bw (  KiB/s): min=  256, max= 1087, per=99.88%, avg=625.25, stdev=304.61, samples=97
   iops        : min=  114, max=  484, avg=278.60, stdev=135.60, samples=97
  lat (usec)   : 4=0.45%, 10=31.05%, 20=11.90%, 50=54.63%, 100=1.83%
  lat (usec)   : 250=0.13%, 500=0.01%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=738, max=229220, avg=3557.53, stdev=3572.11
    sync percentiles (usec):
     |  1.00th=[  824],  5.00th=[  889], 10.00th=[  938], 20.00th=[ 1012],
     | 30.00th=[ 2024], 40.00th=[ 2868], 50.00th=[ 2966], 60.00th=[ 3064],
     | 70.00th=[ 3228], 80.00th=[ 5604], 90.00th=[ 7898], 95.00th=[ 9241],
     | 99.00th=[11863], 99.50th=[13960], 99.90th=[23200], 99.95th=[28705],
     | 99.99th=[91751]
  cpu          : usr=0.41%, sys=2.45%, ctx=48752, majf=0, minf=16
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=626KiB/s (641kB/s), 626KiB/s-626KiB/s (641kB/s-641kB/s), io=29.0MiB (31.5MB), run=49052-49052msec

Disk stats (read/write):
  nvme1n1: ios=0/104595, merge=0/4081, ticks=0/250458, in_queue=308001, util=99.31%
```

(Also, there's no effect of adding or removing `ionice -c 3` basically)

But:

## Concurrent updates with "bfq"

```
+ chrt --idle 0 ionice -c 3 ostree --repo=repo pull-local --fsync-incremental srv-repo/repo bcfae65a2ab0b4ab5afcd1c9f1cce30b300e5408b04fc8bdec51a143ab593d40
+ fio --rw=write --ioengine=sync --fdatasync=1 --directory=etcdtest --size=30m --bs=2300 --name=etcd
7602 metadata, 30681 content objects imported                                                                                                                                                                                                                   

real	1m1.862s
user	0m22.214s
sys	0m17.214s
+ wait
+ cat fio.log
etcd: (g=0): rw=write, bs=(R) 2300B-2300B, (W) 2300B-2300B, (T) 2300B-2300B, ioengine=sync, iodepth=1
fio-3.19
Starting 1 process
etcd: Laying out IO file (1 file / 30MiB)

etcd: (groupid=0, jobs=1): err= 0: pid=1023434: Fri Jul 17 18:43:38 2020
  write: IOPS=426, BW=959KiB/s (982kB/s)(29.0MiB/32040msec); 0 zone resets
    clat (usec): min=2, max=410, avg=23.20, stdev=14.69
     lat (usec): min=2, max=411, avg=23.92, stdev=14.95
    clat percentiles (usec):
     |  1.00th=[    5],  5.00th=[    6], 10.00th=[    7], 20.00th=[   10],
     | 30.00th=[   16], 40.00th=[   21], 50.00th=[   26], 60.00th=[   27],
     | 70.00th=[   29], 80.00th=[   35], 90.00th=[   37], 95.00th=[   41],
     | 99.00th=[   58], 99.50th=[   67], 99.90th=[  114], 99.95th=[  318],
     | 99.99th=[  400]
   bw (  KiB/s): min=  687, max= 1073, per=100.00%, avg=959.86, stdev=65.72, samples=63
   iops        : min=  306, max=  478, avg=427.60, stdev=29.26, samples=63
  lat (usec)   : 4=0.63%, 10=19.43%, 20=19.26%, 50=58.89%, 100=1.67%
  lat (usec)   : 250=0.07%, 500=0.05%
  fsync/fdatasync/sync_file_range:
    sync (usec): min=734, max=97882, avg=2312.55, stdev=1710.08
    sync percentiles (usec):
     |  1.00th=[  840],  5.00th=[  906], 10.00th=[  938], 20.00th=[  996],
     | 30.00th=[ 1057], 40.00th=[ 1156], 50.00th=[ 2900], 60.00th=[ 2999],
     | 70.00th=[ 3097], 80.00th=[ 3195], 90.00th=[ 3294], 95.00th=[ 3392],
     | 99.00th=[ 6194], 99.50th=[ 7439], 99.90th=[10421], 99.95th=[16057],
     | 99.99th=[78119]
  cpu          : usr=0.47%, sys=3.22%, ctx=50926, majf=0, minf=17
  IO depths    : 1=200.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,13677,0,0 short=13677,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=959KiB/s (982kB/s), 959KiB/s-959KiB/s (982kB/s-982kB/s), io=29.0MiB (31.5MB), run=32040-32040msec

Disk stats (read/write):
  nvme1n1: ios=0/39177, merge=0/1428, ticks=0/510678, in_queue=525801, util=99.53%
```
