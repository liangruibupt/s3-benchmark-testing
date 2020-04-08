# The aws s3 cli testing for EC2 and S3 in same region transfer

To optimize performance, we recommend that you access the bucket from Amazon EC2 instances in the same AWS Region when possible. 

- [Multipart Upload support](http://docs.aws.amazon.com/AmazonS3/latest/dev/mpuoverview.html): If the files are over a certain size, the AWS CLI automatically breaks the files into smaller parts and uploads them in parallel. This is done to improve performance and to minimize impact due to network errors. Once all the parts are uploaded, Amazon S3 assembles them into a single object. 

- [AWS S3 CLI Configuration](https://docs.aws.amazon.com/cli/latest/topic/s3-config.html): AWS CLI automatically uses up to 10 threads to upload files or parts to Amazon S3, which can dramatically speed up the upload. You can configure the threads based on your requirements.

[S3 CLI guide](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)


## AWS S3 CLI Configuration
The testing is running on **m4.16xlarge** 64vCPU, 256GiB RAM, 1TB gp2 SSD, 25Gbps network bandwidth
```bash
aws configure --profile cn set s3.max_concurrent_requests 100
aws configure set --profile cn s3.max_queue_size 10000
aws configure set --profile cn s3.multipart_threshold 10MB
aws configure set --profile cn s3.multipart_chunksize 5MB
```

## Test case summary
* [Test case 1: Uploading a large number of very small files to same region Amazon S3](AWS-S3-CLI-Smallfiles-SameRegion.md)
* [Test case 2: Uploading a small number of very large files to same region Amazon S3](AWS-S3-CLI-Bigfiles-SameRegion.md)
* [Test case 3: Uploading large number of small and large files in same region](AWS-S3-CLI-Mixfiles-SameRegion.md)
* [Test case 4: Periodically Synchronizing a Directory in same region](AWS-S3-CLI-SyncTesting-SameRegion.md)

## Conclusion
100 multiple thread + Parallel Streams (run commands for each folder)

Ningxia region

| Case              |  Files     |  Times     |  File Rate    | Throughput    |
| :---------------- | :----------| :--------- | :-----------  | :-----------  |
| Small files 16KB  | 53,248 (1666MiB) |  24 sec   | 2218.6  files/sec | 69.4 MiB/s  | 
| Big files 512MB   | 780 (396863MiB) |  1634 sec | 0.477 files/sec  | 242.8 MiB/s |
| Mix files 512K, 1M, 2M, ... 256M   | 16368 (40961MiB) |  187 sec | 87.5 files/sec  | 219.1 MiB/s |
| Mix files Sync 512K, 1M, 2M, ... 256M   | 16368 (40961MiB) |  169 sec | 96.85 files/sec  | 242.37 MiB/s |


## Resource
[Getting the Most Out of the Amazon S3 CLI](https://aws.amazon.com/blogs/apn/getting-the-most-out-of-the-amazon-s3-cli/)