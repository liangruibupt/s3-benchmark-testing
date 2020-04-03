# The aws s3 cli testing for Cross-Region transfer
## Test case 2: Uploading a small number of very large files to Amazon S3
### 1. s3.max_concurrent_requests 100
```bash
mkdir -p ../bigfile && cd ../bigfile
# Create 50 files filled with 512 MB of pseudo-random content:
seq -w 1 50 | xargs -n1 -P 50 -I % dd if=/dev/urandom of=bigfile.% bs=1024k count=512

# Check the generated files
find . -type f | wc -l
50

du -sm .
25601	.

# Copy the files to AWS Hongkong region S3 bucket
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/ --region ap-east-1
real	2m54.397s
user	7m0.965s
sys	1m12.337s

# capture the number of open connections
# The same as s3.max_concurrent_requests
lsof -i tcp:443 | tail -n +2 | wc -l
100

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
04:37:22 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
04:37:32 PM  all    0.72    0.00    0.12    0.00    0.00    0.04    0.00    0.00    0.00   99.13

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/ --region ap-east-1 | wc -l
50
```

**Summary**:
1. Based on the **real** output of **time** command, it took 2m54s to complete the copy of all directories and the files.
2. The total 50 files at a rate of 50files/174sec=0.287files/sec (25601MiB/174sec=147.1MiB/s) using 100 parallel thread. 
3. The file size is meet the multipart_threshold=10MB, so s3 cli run multipart upload.

### 2. s3.max_concurrent_requests 200
```bash
aws configure set default.s3.max_concurrent_requests 200
# Copy the files to AWS Hongkong region S3 bucket
time aws s3 cp --recursive --quiet . s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/ --region ap-east-1
real	3m0.398s
user	6m40.456s
sys	0m50.434s

# capture the number of open connections
# The same as s3.max_concurrent_requests
lsof -i tcp:443 | tail -n +2 | wc -l
200

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/ --region ap-east-1 | wc -l
50
```

**Summary**:
1. Based on the **real** output of **time** command, it took 3mins to complete the copy of all directories and the files.
2. Increase max_concurrent_requests to 200 has no too much different from max_concurrent_requests to 200. We can test the EC2 and S3 within same region. 
3. The total 50 files at a rate of 50files/180sec=0.277files/sec (25601MiB/180sec=142.2MiB/s) using 200 parallel thread. 
4. The file size is meet the multipart_threshold=10MB, so s3 cli run multipart upload.


### 3. s3.max_concurrent_requests 100 and running multiple commands in parallel to maximize throughput
- xargs launches up to 30 parallel (‘-P30’) invocations of aws s3 cp. Only 26 are actually launched based on the output of the find.

```bash
aws configure set default.s3.max_concurrent_requests 100
cd bigfile
rm bigfile.*
for i in {a..z}; do
    mkdir $i
    seq -w 1 30 | xargs -n1 -P 50 -I % dd if=/dev/urandom of=$i/bigfile.% bs=1024k count=512
done

# Check the generated files
find . -type f | wc -l
780

du -sm .
396863	.

# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P30 -I {} aws s3 cp --recursive --quiet {}/ s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/{}/ --region ap-east-1 )
real	19m38.052s
user	61m50.458s
sys	22m58.539s

## Or using below commands
SECONDS=0
for i in `ls -1 .`; do 
    #nohup aws s3 cp --recursive --quiet ${i}/ s3://molex-mess-migration-hk/zhy-upload/test_smallfiles/ --region ap-east-1 &
    aws s3 cp --recursive --quiet ${i}/ s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/${i}/ --region ap-east-1 &
done
wait
echo $SECONDS

# capture the number of open connections
# equal to max_concurrent_requests 100 * 26
lsof -i tcp:443 | tail -n +2 | wc -l
2600

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
03:46:15 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
03:46:25 PM  all    8.90    0.00    1.36   52.85    0.00    1.58    0.00    0.00    0.00   35.31

# Check the summary of uploaded files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_bigfiles/ --region ap-east-1 | wc -l
780 + 50 = 830
```
**Summary**:
1. Based on the **real** output of **time** command, it took 19m38 to complete the copy of all directories and the files.
2. The total 780 files at a rate of 780 files/ 1178 sec=0.662 files/sec (396863 MiB/ 1178 sec=336.9 MiB/s) using 100 parallel thread. 
3. Running multiple commands in parallel to maximize throughput
4. The file size is meet the multipart_threshold=10MB, so s3 cli run multipart upload.
