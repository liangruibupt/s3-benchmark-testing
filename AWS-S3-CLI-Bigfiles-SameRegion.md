# The aws s3 cli testing for Same Region transfer
## Test case 2: Uploading a small number of very large files to Amazon S3

Reuse the data in [Cross region transfer Test case 2](AWS-S3-CLI-Bigfiles.md)

## s3.max_concurrent_requests 100 and running multiple commands in parallel to maximize throughput
- xargs launches up to 30 parallel (‘-P30’) invocations of aws s3 cp. Only 26 are actually launched based on the output of the find.

```bash
cd s3cli-testing/bigfile
# Check the generated files
find . -type f | wc -l
780

du -sm .
396863	.

# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P30 -I {} aws s3 cp --recursive --quiet {}/ s3://ray-batch-lab/zhy-upload/test_bigfiles/{}/ --region cn-northwest-1 )
real	27m14.006s
user	53m3.897s
sys	23m5.455s


# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 125 - 260

# CPU load
# The CPU load is less than 5%
mpstat -P ALL 10
07:00:07 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
07:00:17 AM  all    2.57    0.00    0.60   48.72    0.00    0.26    0.00    0.00    0.00   47.85

# Check the summary of uploaded files
aws s3 ls --recursive s3://ray-batch-lab/zhy-upload/test_bigfiles/ --region ap-east-1 | wc -l
780 + 50 = 830
```
**Summary**:
1. Based on the **real** output of **time** command, it took 27m14 to complete the copy of all directories and the files.
2. The total 780 files at a rate of 780 files/ 1634 sec=0.477 files/sec (396863 MiB/ 1634 sec=242.8 MiB/s) using 100 parallel thread. 
3. Running multiple commands in parallel to maximize throughput
4. The file size is meet the multipart_threshold=10MB, so s3 cli run multipart upload.
