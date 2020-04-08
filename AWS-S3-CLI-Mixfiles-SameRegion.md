# The aws s3 cli testing for Same Region transfer
## Test case 3: Synchronizing a directory that contains a large number of small and large files

Reuse the data in [Cross region transfer Test case 3](AWS-S3-CLI-Mixfiles.md)

### 3. s3.max_concurrent_requests 100 and running multiple commands in parallel to maximize throughput
- xargs launches up to 16 parallel (‘-P16’) invocations of aws s3 cp. Only 10 are actually launched based on the output of the find.

```bash
cd s3cli-testing/mixfile
# Copy the files to AWS Hongkong region S3 bucket
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 cp --recursive --quiet {}/ s3://ray-batch-lab/zhy-upload/test_mixfiles/{}/ --region cn-northwest-1)
real	3m7.929s
user	10m53.130s
sys	6m45.758s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 80-460

# CPU load
# The CPU load is less than 10%
mpstat -P ALL 10

# Check the summary of uploaded files
aws s3 ls --recursive s3://ray-batch-lab/zhy-upload/test_mixfiles/ --region cn-northwest-1 | wc -l
16368
```
**Summary**:
1. Based on the **real** output of **time** command, it took 2m45s to complete the copy of all directories and the files.
3. The total 16368 files at a rate of 16368 / 187sec = 87.5 files/sec (40961 MiB / 165sec = 219.1 MiB/sec) using 100 parallel thread. 
4. When the file size larger than multipart_threshold=10 MiB, the multipart upload will be applied. 
5. Running multiple commands in parallel to maximize throughput
