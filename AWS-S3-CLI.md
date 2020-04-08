# The aws s3 cli testing for EC2 and S3 in different region transfer

EC2 and S3 in different region transfer, it can also simulate on-primse datacenter virtual machine transfer data to AWS S3

- [Multipart Upload support](http://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html): If the files are over a certain size, the AWS CLI automatically breaks the files into smaller parts and uploads them in parallel. This is done to improve performance and to minimize impact due to network errors. Once all the parts are uploaded, Amazon S3 assembles them into a single object. 

- [AWS S3 CLI Configuration](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html): AWS CLI automatically uses up to 10 threads to upload files or parts to Amazon S3, which can dramatically speed up the upload. You can configure the threads based on your requirements.

[S3 CLI guide](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)


## AWS S3 CLI Configuration
The testing is running on **m4.16xlarge** 64vCPU, 256GiB RAM, 1TB gp2 SSD, 25Gbps network bandwidth
```bash
aws configure set default.s3.max_concurrent_requests 100
aws configure set default.s3.max_queue_size 10000
aws configure set default.s3.multipart_threshold 10MB
aws configure set default.s3.multipart_chunksize 5MB
```

## Test case summary
* [Test case 1: Uploading a large number of very small files to Amazon S3](AWS-S3-CLI-Smallfiles.md)
* [Test case 2: Uploading a small number of very large files to Amazon S3](AWS-S3-CLI-Bigfiles.md)
* [Test case 3: Uploading large number of small and large files](AWS-S3-CLI-Mixfiles.md)
* [Test case 4: Periodically Synchronizing a Directory](AWS-S3-CLI-SyncTesting.md)

## Conclusion
100 multiple thread + Parallel Streams (run commands for each folder)

Ningxia region to Hongkong region

| Case              |  Files     |  Times     |  File Rate    | Throughput    |
| :---------------- | :----------| :--------- | :-----------  | :-----------  |
| Small files 16KB  | 53,248 (1666MiB) |  23 sec   | 2315.2 files/sec | 72.4 MiB/s  | 
| Big files 512MB   | 780 (396863MiB) |  1178 sec | 0.662 files/sec  | 336.9 MiB/s |
| Mix files 512K, 1M, 2M, ... 256M   | 16368 (40961MiB) |  165 sec | 99.2 files/sec  | 248.2 MiB/s |
| Mix files Sync 512K, 1M, 2M, ... 256M   | 16368 (40961MiB) |  116 sec | 141.1 files/sec  | 353.1 MiB/s |
| Mix files Cross Region Sync 512K, 1M, 2M, ... 16368 | 16368 (40961MiB) |  66 sec | 248 files/sec  | 620.6 MiB/s |

Ningxia region to Tokyo region

| Case              |  Files     |  Times     |  File Rate    | Throughput    |
| :---------------- | :----------| :--------- | :-----------  | :-----------  |
| Mix files disable Transfer Acceleration | 16368 （40961MiB)    |  366 sec   | 44.72 files/sec | 111.9 MiB/s  | 
| Mix files disable Transfer Acceleration Parallel Streams | 16368 （40961MiB) |  156 sec | 104.9 files/sec | 262.6 MiB/s  | 
| Mix files enable Transfer Acceleration   | 16368 （40961MiB) |  353 sec | 46.36 files/sec  | 116 MiB/s |
| Mix files enable Transfer Acceleration Parallel Streams | 16368 （40961MiB)  |  91.8 sec | 178.3 files/sec  | 446.2 MiB/s |


## Resource
[Getting the Most Out of the Amazon S3 CLI](https://aws.amazon.com/blogs/apn/getting-the-most-out-of-the-amazon-s3-cli/)