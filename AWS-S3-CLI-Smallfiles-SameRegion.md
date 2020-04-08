# The aws s3 cli testing for Same Region transfer
## Test case 1: Uploading a large number of very small files to Amazon S3

Reuse the data in [Cross region transfer Test case 1](AWS-S3-CLI-Smallfiles.md)

## s3.max_concurrent_requests 100 and running multiple commands in parallel to maximize throughput
- higher currency can not provide good performance and will make some parallel upload failed
- xargs launches up to 30 parallel (‘-P30’) invocations of aws s3 cp. Only 26 are actually launched based on the output of the find.

```bash
cd s3cli-testing/smallfile
# Check the generated files
find . -type f | wc -l
53248

du -sm .
1666	.

# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P30 -I {} aws s3 cp --recursive --quiet {}/ s3://ray-batch-lab/zhy-upload/test_smallfiles/{}/ --region cn-northwest-1 )
real	0m24.273s
user	10m25.369s
sys	5m38.405s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 310

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10

# Check the summary of uploaded files
aws s3 ls --recursive s3://ray-batch-lab/zhy-upload/test_smallfiles/ --region cn-northwest-1 | wc -l
53248
```
**Summary**:
1. Based on the **real** output of **time** command, it took 24 seconds to complete the copy of all directories and the files.
2. The total 53,248 files at a rate of 53,248 filesfiles/24 sec=2218.6 files/sec (1666 MiB/24 sec= 69.4 MiB/s) using 100 parallel thread. 
3. Running multiple commands in parallel to maximize throughput
4. Here is not meet the multipart_threshold=10MB, so no multipart upload.

