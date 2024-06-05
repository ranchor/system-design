# Introduction
Back Of The Envelope Calculations or Ballpark estimations 
# Numbers Crunching
## Data Types to Bytes
General estimates. Depending on the language that implements them, the actual size stored in memory may vary
![](../../../resources/sd/estimation_data_types.png)

## Handling Bytes
### Powers of Two
![](../../../resources/sd/estimation_power_of_two.png)

```
Power           Exact Value         Approx Value        Bytes
---------------------------------------------------------------
7                             128
8                             256
10                           1024   1 thousand           1 KB
16                         65,536                       64 KB
20                      1,048,576   1 million            1 MB
30                  1,073,741,824   1 billion            1 GB
32                  4,294,967,296                        4 GB
40              1,099,511,627,776   1 trillion           1 TB
```

### Byte Conversions
![](../../../resources/sd/estimation_byte_conversions.png)

## Time to Seconds
| Hour       | Day               | Month                     | Year                     |
|------------|-------------------|---------------------------|--------------------------|
| 60∗60=3600 | 3600∗24=86400     | 86400∗30=2592000          | 2592000∗12=31104000      |
| 3600 secs  | approx. ~100k secs | approx. ~2.5 million secs | approx. ~30 million secs |

## Requests Calculations
| Requests               | Requests per second |
|------------------------|---------------------|
| 1 million reqs/month   | .4 reqs/sec         |
| 2.5 million reqs/month | 1 reqs/sec          |
| 5 million reqs/month   | 2 reqs/sec          |
| 10 million reqs/month  | 4 reqs/sec          |
| 100 million reqs/month | 40 reqs/sec         |
| 1 billion reqs/month   | 400 reqs/sec        |

## Latency numbers every programmer should know
![](../../../resources/sd/estimation_latency.png)

### Time Conversions
* 1 ns (nanoseconds) = 10^-9 seconds
* 1 μs (microseconds) = 10^-6 seconds = 1,000 ns
* 1 ms (milliseconds) = 10^-3 seconds = 1,000 us = 1,000,000 ns
* 1 s (seconds) = 10¹ = 1,000 ms

**Based on the metrics above you can:**
* Read sequentially from HDD at 30 MB/s
* Read sequentially from 1 Gbps Ethernet at 100 MB/s
* Read sequentially from SSD at 1 GB/s
* Read sequentially from main memory at 4 GB/s
* Make 6–7 world-wide round trips per second
* Make 2,000 round trips per second within a datacenter

## Availability Numbers 
* High availability is measured as a percentage. Most services fall between 99% and 100%.
* A service level agreement (SLA) is a commonly used term for service providers
* Uptime is traditionally measured in nines. The more the nines, the better.
  ![](../../../resources/sd/estimation_availability.png)

## Typical Modern Resources Sizes
### General Purpose Amazon EC2 Instances
* m6g.medium, 4 GiB of Memory, 1 vCPU, EBS only, 64-bit Arm platform
* m6g.16xlarge, 256 GiB of Memory, 64 vCPUs, EBS only, 64-bit Arm platform
* Custom built AWS Graviton2 Processor with 64-bit Arm Neoverse cores
—Where 1 hertz = 1 cycle per second
—2.5GHZ (Gigahertz) — two billion cycles per second
* Support for Enhanced Networking with Up to 25 Gbps of Network bandwidth
### Compute Optimised Amazon EC2 Instances
* c5a.24xlarge, 192 GiB of Memory, 96 vCPUs, EBS only, 64-bit platform
### Memory Optimised Amazon EC2 Instances
* r6g.16xlarge, 512 GiB of Memory, 64 vCPUs, EBS only, 64-bit Arm platform
* x1.32xlarge: 1,952 GiB of memory, 128 vCPUs, 2 x 1,920 GB of SSD-based instance storage, 64-bit platform, 20 
  Gigabit Ethernet
### Storage Optimised Amazon EC2 Instances
* i3en.large: 16 GiB of memory, 2 vCPUs, 1 x 1.25TB NVMe SSD, 64-bit platform
* i3en.24xlarge: 768 GiB of memory, 96 vCPUs, 8 x 7.5TB NVMe SSD, 64-bit platform
* Connections — about 1000 concurrent connections

# References
* https://stormanning.medium.com/the-system-design-interview-part-3-back-of-the-envelope-calculations-8e40793f48a2
* https://www.rkenmi.com/posts/back-of-the-envelope
* [J. Dean.Google Pro Tip - Use Back-Of-The-Envelope-Calculations To Choose The Best Design](http://highscalability.com/blog/2011/1/26/google-pro-tip-use-back-of-the-envelope-calculations-to-choo.html)
* [system-design-primer - Powers of two table](https://github.com/donnemartin/system-design-primer/blob/master/README.md#powers-of-two-table)
* [system-design-primer - Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer/blob/master/README.md#latency-numbers-every-programmer-should-know)
* https://colin-scott.github.io/personal_website/research/interactive_latency.html
* https://matthewdbill.medium.com/back-of-envelope-calculations-cheat-sheet-d6758d276b05