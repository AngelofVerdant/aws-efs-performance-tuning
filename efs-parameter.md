How the number of threads used to write data affects Amazon EFS performance
===========================================================================

1. Enter the following command to set an environment variable named directory that contains the instance ID of this instance:
   
```
directory=$(wget -q -O - http://169.254.169.254/latest/meta-data/instance-id)
```

Note: The Instance ID is used to identify a unique directory on the EFS file share for each instance 

3. Enter the following command to use touch with a single thread to generate 1024 zero-byte files in the /mnt/efs/01/touch directory:

```
time for i in {1..1024}; do touch /mnt/efs/touch/${directory}/test-1.3-$i; done;
```

3. Enter the following command to use touch with multiple threads to generate 1024 zero-byte files in the /mnt/efs/01/touch directory.

```
time seq 1 1024 | parallel --will-cite -j 128 touch /mnt/efs/touch/${directory}/test-1.4-{}
```

4.  Enter the following command to create 32 directories, labeled 1 through 32, in the /mnt/efs/01/touch directory.

```
mkdir -p /mnt/efs/touch/${directory}/{1..32}
```
5.Enter the following command to use touch with multiple threads to generate 1024 zero-byte files in the numbered directories you created in the previous task:

```
time seq 1 32 | parallel --will-cite -j 32 touch /mnt/efs/touch/${directory}/{}/test-1.5-{1..32}
```

How the network performance of EC2 instance types affect Amazon EFS performance
===============================================================================

For this task, you connect to the same EC2 instance using two separate SSM sessions so that you can monitor the network traffic using nload and view the results of the commands you enter at the same time.

Copy the URL of the open SSM connection page you are currently on, and then paste that link into a new browser window. You should now have two SSM sessions open to Performance Instance 1.

 In the first SSM session window, enter the following command to launch nload to monitor network throughput:

 ```
 nload -u M
 ```
In the second SSM session window, enter the following command to use the dd utility to write 10GB of data to the EFS file system:

```
time dd if=/dev/zero of=/mnt/efs/dd/10G-dd-$(date +%Y%m%d%H%M%S.%3N) bs=1M count=10000 conv=fsync
```

- The following list explains the various options you just used with the dd command:

- time tracks how long the command takes to run to completion.
- if= defines the input file to read from.
- of= defines the file to write to.
- bs= defines the number of bytes to read and write at a time.
- count= specifies the the number of blocks to write.
- conv=fsync specifies that metedata should be written as well.
- The end result of the command is that 10,000 1MB blocks of data are written to the EFS file share for a total of 10GB.

N/B:

All EC2 instance types have different network performance characteristics, so each can drive different levels of throughput to Amazon EFS. While the t3.micro instance initially appears to have better network performance when compared to an m4.large instance, its high network throughput is short lived as a result of the burst characteristics of t3 instances. Review the EC2 instance type characteristics for the t3.micro, m4.large and m5.xlarge instances here. In particular, look at the network performance statistics for the different instance types.


 How I/O size and sync frequency affect throughput to Amazon EFS
 ===============================================================

Enter the following command to use dd to create 2 GB of files on the EFS file system using a 1 MB block size and issue a sync once after each file to verify the file is written to disk:

```
time dd if=/dev/zero of=/mnt/efs/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) bs=1M count=2048 status=progress conv=fsync
```

Now Enter the following command to use dd to create 2 GB of files on the EFS file system using a 16 MB block size and issue a sync once after each file to verify the file is written to disk:

```
time dd if=/dev/zero of=/mnt/efs/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) bs=16M count=128 status=progress conv=fsync
```


Now enter the following command to use dd to create 2 GB of files on the EFS file system using a 1 MB block size and issue a sync after each block to verify each block is written to disk:

```
time dd if=/dev/zero of=/mnt/efs/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) bs=1M count=2048 status=progress oflag=sync
```
And Finally , Enter the following command to use dd to create 2 GB of files on the EFS file system using a 16 MB block size and issues a sync after each block to verify each block is written to disk.

```
time dd if=/dev/zero of=/mnt/efs/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N) bs=16M count=128 status=progress oflag=sync
```

Notice that the throughput drops dramatically when the sync method is changed from per-file to per-block. Throughput drops even further when you sync per-block, and the block size is reduced from 16 MB to 1 MB. However, when syncing per-file the block size has little impact on throughput. 


How multi-threaded access improves throughput and IOPS
======================================================

Sync writes to disk any data buffered in memory. This can include (but is not limited to) modified superblocks, modified inodes, and delayed reads and writes. This must be implemented by the Linux kernel. The sync program does nothing but exercise the sync(2) system call.


Enter the following command to use dd to write 2 GB of data to Amazon EFS using a 1 MB block size and 4 threads and issue a sync after each block to ensure everything is written to disk:

```
time seq 0 3 | parallel --will-cite -j 4 dd if=/dev/zero of=/mnt/efs/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{} bs=1M count=512 oflag=sync
```
Now enter the following command to use dd to write 2 GB of data to Amazon EFS using a 1 MB block size and 16 threads and issue a sync after each block to ensure everything is written to disk:

```
time seq 0 15 | parallel --will-cite -j 16 dd if=/dev/zero of=/mnt/efs/dd/2G-dd-$(date +%Y%m%d%H%M%S.%3N)-{} bs=1M count=128 oflag=sync
```

N/B:

The distributed data storage design of EFS means that multi-threaded applications can drive substantial levels of aggregate throughput and IOPS. If you parallelize your writes to EFS by increasing the number of threads, you can increase the overall throughput and IOPS to EFS.


 Compare file transfer tools
 ==========================

Let us explore the way that different file transfer tools affect performance when accessing an Amazon EFS file system.

Enter the following command to monitor the network throughput of the instance:

```
nload -u M
```

For this test,let's connect to the same EC2 instance using two separate SSM sessions so that you can monitor the network traffic using nload and view the results of the commands you enter at the same time. In the second SSM session, enter the following command to create an environment variable named instance_id.

```
instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
```

Enter the following commands to drop caches and use rsync to transfer 5,000 files that are approximately 1 MB each, totaling 5 GB, from instance’s local EBS volume to the EFS file system:

```
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time rsync -r /ebs/data-1m/ /mnt/efs/rsync/${instance_id}
```

Linux kernels 2.6.16 and newer provide a mechanism to have the kernel drop the page cache and/or inode and dentry caches on command, which can help free up a lot of memory. This is a non-destructive operation and only frees things that are completely unused. Dirty objects continue to be in use until written out to disk and are not able to be freed by this mechanism. Clearing cache frees RAM, but it causes the kernel to look for files on the disk rather than in the cache. You can drop caches like this as a method of benchmarking disk performance.

Now enter the following commands to drop caches and use cp to transfer 5,000 files of approximately 1 MB each, totaling 5 GB, from the instance’s local EBS volume to the EFS file system:

```
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time cp -r /ebs/data-1m/* /mnt/efs/cp/${instance_id}
```
 Now enter the following command to set the threads environment variable to 4 threads per vCPU. The variable is used in upcoming steps. As the m5.xlarge instance has 8 vCPU, the command enables the use of 32 threads, which is the number of vCPU x 4).

 ```
threads=$(($(nproc --all) * 4))
```

Now  enter the following commands to drop caches and use fpsync to transfer 5,000 files approximately 1 MB each, totaling 5 GB, from the instance’s EBS volume to the EFS file system:


```
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time /usr/local/bin/fpsync -n ${threads} -v /ebs/data-1m/ /mnt/efs/fpsync/${instance_id}

```

And then run following commands to drop caches and use cp and GNU Parallel to transfer 5,000 files approximately 1 MB each, totaling 5 GB, from the instance’s EBS volume to the EFS file system:

```
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
time find /ebs/data-1m/. -type f | parallel --will-cite -j ${threads} cp {} /mnt/efs/parallelcp

```

Enter the following commands to drop caches and use fpart, cpio, and GNU Parallel to transfer 5,000 files approximately 1 MB each, totaling 5 GB, from the instance’s EBS volume to the EFS file system:

```
sudo su
sync && echo 3 > /proc/sys/vm/drop_caches
exit
cd /ebs/smallfile
time /usr/local/bin/fpart -z -n 1 -o /home/ec2-user/fpart-files-to-transfer .
head /home/ec2-user/fpart-files-to-transfer.0
time parallel --will-cite -j ${threads} --pipepart --round-robin --delay .1 --block 1M -a /home/ec2-user/fpart-files-to-transfer.0 sudo "cpio -dpmL /mnt/efs/parallelcpio/${instance_id}"

```
Amazon EFS file systems are distributed across an unconstrained number of storage servers and this distributed data storage design means that multithreaded applications like fpsync, mcp, and GNU parallel can drive substantial levels of throughput and IOPS to EFS when compared to single-threaded applications.























