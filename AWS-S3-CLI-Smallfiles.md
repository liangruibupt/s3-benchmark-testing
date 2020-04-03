# The aws s3 cli testing for Cross-Region transfer
## Test case 1: Uploading a large number of very small files to Amazon S3
### 1. s3.max_concurrent_requests 100
```bash
mkdir -p s3cli-testing && cd s3cli-testing
# Create the 26 directories named for each letter of the alphabet, then create 2048 files containing 32K of pseudo-random content in each
mkdir -p smallfile && cd smallfile
for i in {a..z}; do
    mkdir $i
    seq -w 1 2048 | xargs -n1 -P 256 -I % dd if=/dev/urandom of=$i/% bs=32k count=1
done

# Check the generated files
find . -type f | wc -l
53248

du -sm .
1666	.

# Copy the files to AWS Hongkong region S3 bucket
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1
real	5m50.230s
user	6m51.454s
sys	1m0.817s

# capture the number of open connections
# The same as s3.max_concurrent_requests
lsof -i tcp:443 | tail -n +2 | wc -l
100

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
04:12:27 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
04:12:37 PM  all    1.27    0.00    0.17    0.00    0.00    0.01    0.00    0.00    0.00   98.55

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 | wc -l
53248
```

**Summary**:
1. Based on the **real** output of **time** command, it took 5m50s to complete the copy of all directories and the files.
2. The total 53,248 files at a rate of 152 files/sec (1666MiB/350sec = 4.76MiB/sec) using 100 parallel thread. 
3. Here is not meet the multipart_threshold=10MB, so no multipart upload.

### 2. s3.max_concurrent_requests 200
```bash
aws configure set default.s3.max_concurrent_requests 200

# Copy the files to AWS Hongkong region S3 bucket
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1
real	5m13.574s
user	7m5.473s
sys	1m7.049s

# capture the number of open connections
# The same as s3.max_concurrent_requests
lsof -i tcp:443 | tail -n +2 | wc -l
NOT stable, can change from 70-140

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
08:58:13 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
08:58:23 AM  all    1.25    0.00    0.20    0.03    0.00    0.02    0.00    0.00    0.00   98.51

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 | wc -l
53248
```
**Summary**:
1. Based on the **real** output of **time** command, it took 5m13s to complete the copy of all directories and the files.
2. Increase max_concurrent_requests to 200 can reduce time consuming, but not significant. 
3. The total 53,248 files at a rate of 53,248/313sec = 170 files/sec (1666MiB/313sec = 5.32MiB/sec) using 200 parallel thread. 
4. Here is not meet the multipart_threshold=10MB, so no multipart upload.

### 3. s3.max_concurrent_requests 100 and running multiple commands in parallel to maximize throughput
- higher currency can not provide good performance and will make some parallel upload failed
- xargs launches up to 30 parallel (‘-P30’) invocations of aws s3 cp. Only 26 are actually launched based on the output of the find.

```bash
aws configure set default.s3.max_concurrent_requests 100
cd smallfile
# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P30 -I {} aws s3 cp --recursive --quiet {}/ s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/{}/ --region ap-east-1 )
real	0m22.857s
user	10m22.527s
sys	3m57.305s

## Or using below commands
SECONDS=0
for i in `ls -1 .`; do 
    #nohup aws s3 cp --recursive --quiet ${i}/ s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 &
    aws s3 cp --recursive --quiet ${i}/ s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/${i}/ --region ap-east-1 &
done
wait
echo $SECONDS
output: 23

# capture the number of open connections
# equal to max_concurrent_requests 100 * 26
lsof -i tcp:443 | tail -n +2 | wc -l
2600

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
08:58:13 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
08:58:23 AM  all    1.25    0.00    0.20    0.03    0.00    0.02    0.00    0.00    0.00   98.51

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 | wc -l
53248
```
**Summary**:
1. Based on the **real** output of **time** command, it took 23 seconds to complete the copy of all directories and the files. It is big improvement compared with first 5m50s transfer.
2. The total 53,248 files at a rate of 53,248 filesfiles/23 sec=2315.2 files/sec (1666 MiB/23 sec= 72.4 MiB/s) using 100 parallel thread. 
3. Running multiple commands in parallel to maximize throughput
4. Here is not meet the multipart_threshold=10MB, so no multipart upload.

