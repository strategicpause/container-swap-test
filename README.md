# Introduction
The purpose of this project is to help me better understand usage of swap memory in containers and how it affects both
container performance and the ability to run certain workloads.

For this experiment I'm using sysbench memory write benchmark. Sysbench is ran in a container via podman, which is used 
to configure the memory properties of the container; this includes the amount of anonymous and swap memory available to 
the container. 

# Setup
Software versions
~~~
$ uname --all
Linux fedora 6.4.13-100.fc37.x86_64 #1 SMP PREEMPT_DYNAMIC Wed Aug 30 16:43:51 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

$ podman --version
podman version 4.6.1

$ sysbench --version
sysbench 1.0.20
~~~

his was ran on 12 vCPU (6-core with 2 hyper-threads) AMD Ryzen 5 3600 with 16 GiB memory. The disk is a 1 TB (931 GiB)
SATA SSD drive.

# Experiment Setup
## Build the container
~~~
$ podman build . -t sysbench    
~~~

## Run the Test
~~~
$ podman run \
    --memory <1>m \
    --memory-swap <2>m \ 
    --name=sysbench \
    --rm \ 
    localhost/sysbench \ 
    sysbench memory --memory-block-size=<3>G --memory-total-size=<4>G --memory-oper=write run
~~~

1. **Memory** - This represents the amount of anonymous memory available to the container.
2. **Memory-Swap** - The amount of swap memory available to the container. Setting this value to 0 will turn off swap 
memory.
3. **Memory-Block-Size** - This is the size of the buffer used for write operations in the sysbench test. 
4. **Memory-Total-Size** - How much memory is written during the test. This is set to 20 GiB for all test.

Optionally `--cgroup-conf` is used to experiment with `memory.high` which was introduced in cgroupsv2 to throttle
memory usage limit. If memory in the cgroup goes above this value, then the processes are throttled and put under heavy
reclaim pressure - in practice I've observed that anonymous memory will not go beyond this level and will instead
use any available swap memory.

### Examples

~~~
$ podman run \
    --memory 4096m \
    --memory-swap 0m \ 
    --name=sysbench \
    --rm \ 
    localhost/sysbench \ 
    sysbench memory --memory-block-size=4G --memory-total-size=20G --memory-oper=write run
~~~
Run the test with 4096 MiB - no swap

~~~
$ podman run \
    --memory 3072m \
    --memory-swap 4096m \ 
    --name=sysbench \
    --rm \ 
    localhost/sysbench \ 
    sysbench memory --memory-block-size=4G --memory-total-size=20G --memory-oper=write run
~~~
Run the test with 3072 MiB - 1 GiB of swap. Note that memory-swap must be greater than the configured memory value.

~~~
$ podman run \
    --memory 4096m \
    --memory-swap 4196m \ 
    --name=sysbench \
    --rm \
    --cgroup-conf memory.high=4294967296 \ 
    localhost/sysbench \
    sysbench memory --memory-block-size=1G --memory-total-size=20G --memory-oper=write run
~~~
Usage of Memory.High

# Results
| Container Memory (MiB) | Container Swap (MiB) | Memory.High (MiB) | Sysbench Block Size | Sysbench Total Size | Avg (ms) | 95th (ms) | Sum (ms) |
|------------------------|----------------------|-------------------| ------------------- | ------------------- | -------- | --------- | -------- |
| 3072                   | 4096                 | 0                 | 1G                  | 20G                 | 107      | 108       | 2,147    |
| 4096                   | 4196                 | 4096              | 1G                  | 20G                 | 107      | 110       | 2,145    |
| 4096                   | 5120                 | 0                 | 1G                  | 20G                 | 105      | 110       | 2,167    |
| 4096                   | 0                    | 0                 | 1G                  | 20G                 | 108      | 110       | 2,168    |
| 4096                   | 4196                 | 0                 | 1G                  | 20G                 | 111      | 123       | 2,239    |
| 3840                   | 4196                 | 0                 | 1G                  | 20G                 | 109      | 127       | 2,198    |
| 4196                   | 0                    | 0                 | 1G                  | 20G                 | 112      | 127       | 2,242    |
| 3072                   | 0                    | 0                 | 1G                  | 20G                 | 112      | 137       | 2,241    |
| 4096                   | 5120                 | 0                 | 2G                  | 20G                 | 217      | 227       | 2,170    |
| 4096                   | 0                    | 0                 | 2G                  | 20G                 | 215      | 231       | 2,157    |
| 4096                   | 4196                 | 4096              | 2G                  | 20G                 | 219      | 235       | 2,193    |
| 3072                   | 4096                 | 0                 | 2G                  | 20G                 | 219      | 244       | 2,193    |
| 3840                   | 4196                 | 0                 | 2G                  | 20G                 | 222      | 257       | 2,223    |
| 4196                   | 0                    | 0                 | 2G                  | 20G                 | 226      | 257       | 2,266    |
| 4096                   | 4196                 | 0                 | 2G                  | 20G                 | 222      | 267       | 2,229    |
| 3072                   | 0                    | 0                 | 2G                  | 20G                 | 221      | 272       | 2,215    |
| 4196                   | 0                    | 0                 | 4G                  | 20G                 | 435      | 458       | 2,179    |
| 4096                   | 4196                 | 0                 | 4G                  | 20G                 | 6,061    | 7,479     | 12,123   |
| 4096                   | 4196                 | 4096              | 4G                  | 20G                 | 6,565    | 8,638     | 13,130   |
| 4096                   | 5120                 | 0                 | 4G                  | 20G                 | 11,696   | 15,371    | 23,393   |
| 3072                   | 4196                 | 3072              | 4G                  | 20G                 | 12,594   | 15,934    | 25,188   |
| 3840                   | 4196                 | 0                 | 4G                  | 20G                 | 12,232   | 16,224    | 24,464   |
| 3072                   | 4196                 | 0                 | 4G                  | 20G                 | 12,644   | 16,224    | 25,289   |
| 4096                   | 0                    | 0                 | 4G                  | 20G                 | \-       | \-        | \-       |
| 3072                   | 0                    | 0                 | 4G                  | 20G                 | \-       | \-        | \-       |
| 3072                   | 4096                 | 0                 | 4G                  | 20G                 | \-       | \-        | \-       |

# Observations
* When sizing the container process overhead must be considered in addition to the the buffer size.
* Usage of swap memory results in times that are orders of magnitude higher than experiments with no swap usage (even
when swap is enabled).
* There are no major performance differences when using 1GB and 2GB buffers - this is likely because swap and 
memory.high comes into play.
* Using `Memory.High` seems to result in poor results when hitting memory limitations - even in cases when swap is 
enabled.
