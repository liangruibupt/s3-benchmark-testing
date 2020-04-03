# Enable S3 Transfer Acceleration to improve upload and download performance

EC2 and S3 in different region transfer, it can also simulate on-primse datacenter virtual machine transfer data to AWS S3

- [S3 Transfer Acceleration](https://docs.aws.amazon.com/AmazonS3/latest/dev/transfer-acceleration.html)
- [Accelerate speed comparsion](http://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html)


## 3.1 Enabling Transfer Acceleration on a Bucket
Use the Tokyo region S3 bucket which support `Transfer Acceleration` feature
```bash
aws s3api put-bucket-accelerate-configuration --bucket molex-mess-migration-tokyo --accelerate-configuration Status=Enabled --region ap-northeast-1
Transfer Acceleration endpoint: molex-mess-migration-tokyo.s3-accelerate.amazonaws.com
```

## 3.2 Tuning multiple threads and multiple part for AWS CLI
```bash
# Tuning multiple threads and multiple part
aws configure set default.s3.max_concurrent_requests 100
aws configure set default.s3.max_queue_size 10000
aws configure set default.s3.multipart_threshold 10MB
aws configure set default.s3.multipart_chunksize 5MB
```

## 3.3 Upload 40GB files by S3 without Transfer Acceleration
Test the scenario as [Test case 3: Uploading large number of small and large files](AWS-S3-CLI-Mixfiles.md)
1. Disable Transfer Acceleration
```bash
cd mixfile

# Check the generated files
find . -type f | wc -l
16368

du -sm .
40961	.

# Disable Transfer Acceleration
aws configure set default.s3.use_accelerate_endpoint false

# Copy the files to AWS Tokyo region S3 bucket directly
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-tokyo/zhy-upload/test_mixfiles/ --region ap-northeast-1
real	6m6.797s
user	14m3.135s
sys	1m38.637s

# capture the number of open connections
# s3.max_concurrent_requests 100
lsof -i tcp:443 | tail -n +2 | wc -l
80-100

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-tokyo/zhy-upload/test_mixfiles/ --region ap-northeast-1 | wc -l
16368
```

2. Disable Transfer Acceleration + Running multiple commands in parallel to maximize throughput
```bash
# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 cp --recursive --quiet {}/ s3://molex-mess-migration-tokyo/zhy-upload/test_mixfiles/{}/ --region ap-northeast-1 )
real	2m36.371s
user	8m9.641s
sys	3m24.312s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 963, less than max_concurrent_requests * 10 = 1000

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-tokyo/zhy-upload/test_mixfiles/ --region ap-northeast-1 | wc -l
16368
```

## 3.4 Upload 40GB files by S3 Transfer Acceleration
1. Enable Transfer Acceleration
```bash
# Enable Transfer Acceleration
aws configure set default.s3.use_accelerate_endpoint true
aws configure set s3.addressing_style virtual

# Copy the files to AWS Tokyo region S3 bucket directly
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-tokyo/transfer-acc/test_mixfiles/ --region ap-northeast-1
# OR you can use
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-tokyo/transfer-acc/test_mixfiles/ --region ap-northeast-1 --endpoint-url http://s3-accelerate.amazonaws.com
real	5m53.382s
user	13m21.686s
sys	1m41.201s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
0

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-tokyo/transfer-acc/test_mixfiles/ --region ap-northeast-1 | wc -l
16368
```

2. Enable Transfer Acceleration + Running multiple commands in parallel to maximize throughput
```bash
# Disable Transfer Acceleration
aws configure set default.s3.use_accelerate_endpoint true

# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 cp --recursive --quiet {}/ s3://molex-mess-migration-tokyo/transfer-acc/test_mixfiles/{}/ --region ap-northeast-1 )
# OR you can use
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 cp --recursive --quiet {}/ s3://molex-mess-migration-tokyo/transfer-acc/test_mixfiles/{}/ --region ap-northeast-1 --endpoint-url http://s3-accelerate.amazonaws.com)
real	1m31.827s
user	10m19.633s
sys	4m19.812s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
0

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-tokyo/transfer-acc/test_mixfiles/ --region ap-northeast-1 | wc -l
16368
```

## Summary
100 multiple thread + Parallel Streams (run commands for each folder)

| Case              |  Files     |  Times     |  File Rate    | Throughput    |
| :---------------- | :----------| :--------- | :-----------  | :-----------  |
| Mix files disable Transfer Acceleration | 16368 （40961MiB)    |  366 sec   | 44.72 files/sec | 111.9 MiB/s  | 
| Mix files disable Transfer Acceleration Parallel Streams | 16368 （40961MiB) |  156 sec | 104.9 files/sec | 262.6 MiB/s  | 
| Mix files enable Transfer Acceleration   | 16368 （40961MiB) |  353 sec | 46.36 files/sec  | 116 MiB/s |
| Mix files enable Transfer Acceleration Parallel Streams | 16368 （40961MiB)  |  91.8 sec | 178.3 files/sec  | 446.2 MiB/s |
