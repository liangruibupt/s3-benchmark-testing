# The aws s3 cli testing for Cross-Region transfer
## Test case 3: Synchronizing a directory that contains a large number of small and large files
### 1. s3.max_concurrent_requests 100
```bash
aws configure set default.s3.max_concurrent_requests 100

# Create a mix of file sizes from 512K, 1M, 2M, ... 256M, 
mkdir -p mixfile && cd mixfile

i=1;
while [[ $i -le 512 ]]; do
    num=$((8192/$i))
    [[ $num -ge 1 ]] || num=1
    mkdir -p $i
    seq -w 1 $num | xargs -n1 -P 256 -I % dd if=/dev/urandom of=$i/mixfile_$i.% bs=512k count=$i;
    i=$(($i*2))
done

# Check the generated files
find . -type f | wc -l
16368

du -sm .
40961	.

# Copy the files to AWS Hongkong region S3 bucket
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-hk/zhy-upload/test_mixfiles/ --region ap-east-1
real	6m29.426s
user	12m30.636s
sys	1m55.983s

# capture the number of open connections
The same as s3.max_concurrent_requests 100
lsof -i tcp:443 | tail -n +2 | wc -l
100

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
04:23:47 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
04:23:57 PM  all    1.10    0.00    0.18    0.46    0.00    0.10    0.00    0.00    0.00   98.17

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_mixfiles/ --region ap-east-1 | wc -l
16368
```

**Summary**:
1. Based on the **real** output of **time** command, it took 6m29s to complete the copy of all directories and the files.
2. The total 16368 files at a rate of 16368 / 389sec = 42.1 files/sec (40961 MiB / 389sec = 105 MiB/sec) using 100 parallel thread. 
3. When the file size larger than multipart_threshold=10 MiB, the multipart upload will be applied.

### 2. s3.max_concurrent_requests 200
```bash
aws configure set default.s3.max_concurrent_requests 200

# Copy the files to AWS Hongkong region S3 bucket
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-hk/zhy-upload/test_mixfiles/ --region ap-east-1
real	6m23.988s
user	12m56.511s
sys	2m19.205s

# capture the number of open connections
# The same as s3.max_concurrent_requests
lsof -i tcp:443 | tail -n +2 | wc -l
NOT stable, can change from 70-140

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
04:38:06 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
04:38:16 PM  all    1.36    0.00    0.24    0.00    0.00    0.06    0.00    0.00    0.00   98.34

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_mixfiles/ --region ap-east-1 | wc -l
16368
```
**Summary**:
1. Based on the **real** output of **time** command, it took 6m23s to complete the copy of all directories and the files.
2. Increase max_concurrent_requests to 200 can reduce time consuming, but not significant. 
3. The total 16368 files at a rate of 16368 / 383sec = 42.7 files/sec (40961 MiB / 383sec = 106.9 MiB/sec) using 100 parallel thread. 
4. When the file size larger than multipart_threshold=10 MiB, the multipart upload will be applied.

### 3. s3.max_concurrent_requests 100 and running multiple commands in parallel to maximize throughput
- xargs launches up to 16 parallel (‘-P16’) invocations of aws s3 cp. Only 10 are actually launched based on the output of the find.

```bash
aws configure set default.s3.max_concurrent_requests 100

cd mixfile
# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 cp --recursive --quiet {}/ s3://molex-mess-migration-hk/zhy-upload/test_mixfiles/{}/ --region ap-east-1 )
real	2m45.275s
user	8m34.664s
sys	3m28.710s

## Or using below commands
SECONDS=0
for i in `ls -1 .`; do 
    #nohup aws s3 cp --recursive --quiet ${i}/ s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 &
    aws s3 cp --recursive --quiet ${i}/ s3://molex-mess-migration-hk/zhy-upload/test_mixfiles/${i}/ --region ap-east-1 &
done
wait
echo $SECONDS

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 959, less than max_concurrent_requests * 10 = 1000

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
05:02:49 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
05:02:59 PM  all    2.63    0.00    0.70    0.00    0.00    0.65    0.00    0.00    0.00   96.02

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 | wc -l
16368
```
**Summary**:
1. Based on the **real** output of **time** command, it took 2m45s to complete the copy of all directories and the files.
3. The total 16368 files at a rate of 16368 / 165sec = 198.6 files/sec (40961 MiB / 165sec = 248.2 MiB/sec) using 100 parallel thread. 
4. When the file size larger than multipart_threshold=10 MiB, the multipart upload will be applied. 
5. Running multiple commands in parallel to maximize throughput
