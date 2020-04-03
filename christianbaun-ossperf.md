# OSSperf benchmark tool

OSSperf is a lightweight command-line tool for analyzing the performance and data integrity of storage services that implement the S3 API.

The tool creates a user-defined number of files with random content and of a specified size inside a local directory. And benchmark the performance of GET, PUT and DELETE 

https://github.com/christianbaun/ossperf

## prepare
```bash
export AWS_ACCESS_KEY_ID=<ACCESSKEY>
export AWS_SECRET_ACCESS_KEY=<SECRETKEY>

# install parallel
sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
sudo yum install parallel nload -y

# install s3cmd
sudo yum install -y s3cmd

# Configure the s3 cli
aws configure set default.s3.max_concurrent_requests 100
aws configure set default.s3.max_queue_size 10000
aws configure set default.s3.multipart_threshold 10MB
aws configure set default.s3.multipart_chunksize 5MB
```

## Testing
1. Testing 1000 small files with each 16KiB to Hongkong region S3 bucket
```bash
# aws s3 cli
./ossperf.sh -n 1000 -s 16384 -b ossperf-molex-mess-migration-hongkong -p -w ap-east-1

[1] Required time to create the bucket:                 2.657 s
[2] Required time to upload the files:                  24.455 s
[3] Required time to fetch a list of files:             2.442 s
[4] Required time to download the files:                31.331 s
[5] Required time to erase the objects:                 21.252 s
[6] Required time to erase the bucket:                  1.624 s
    Required time to perform all S3-related operations: 83.761 s

    Bandwidth during the upload of the files:           5.359 Mbps
    Bandwidth during the download of the files:         4.183 Mbps

lsof -i tcp:443 | tail -n +2 | wc -l
# less than 4 * 100 (s3.max_concurrent_requests)
100
```

2. Testing 1000 files with each 16MiB to Hongkong region S3 bucket
```bash
./ossperf.sh -n 1000 -s 16777216 -b ossperf-molex-mess-migration-hongkong -p -w ap-east-1

[1] Required time to create the bucket:                 2.395 s
[2] Required time to upload the files:                  80.314 s
[3] Required time to fetch a list of files:             2.440 s
[4] Required time to download the files:                545.128 s
[5] Required time to erase the objects:                 22.053 s
[6] Required time to erase the bucket:                  1.386 s
    Required time to perform all S3-related operations: 653.716 s

    Bandwidth during the upload of the files:           1671.162 Mbps
    Bandwidth during the download of the files:         246.213 Mbps

lsof -i tcp:443 | tail -n +2 | wc -l
# less than 4 * 100 (s3.max_concurrent_requests)
185 - 340
```

## Conclusion
100 multiple thread + Parallel Streams (run commands for each folder)

| Case              |  Files     |  Upload Times  |  Upload Throughput  |  Download Times  |  Download Throughput  |
| :---------------- | :----------| :------------- | :-----------------  | :--------------  | :-------------------- |
| Small files 16KB  | 1000    |  24.46 sec   | 0.67 MiB/s | 31.331 sec  | 0.52 MiB/s  | 
| Files 16MiB       | 1000    |  80.3 sec   | 208.9 MiB/s | 545.128 sec | 30.78 MiB/s  | 
