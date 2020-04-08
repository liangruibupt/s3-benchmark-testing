# The aws s3 cli testing for Cross-Region transfer
## Test case 4: Periodically Synchronizing a Directory

* [Reuse the Cross region transfer Test case 4](AWS-S3-CLI-SyncTesting.md)

## Parallel Streams with multiple threads
```bash
# - xargs launches up to 16 parallel (‘-P16’) invocations of aws s3 cp. Only 10 are actually launched based on the output of the find.
cd s3cli-testing/randfiles
i=1;
while [[ $i -le 512 ]]; do
    touch $i/*
    i=$(($i*2))
done
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 sync --quiet {}/ s3://ray-batch-lab/zhy-upload/test_randfiles/{}/ --region cn-northwest-1 )
real	2m49.699s
user	9m27.695s
sys	5m18.966s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 80-450

# CPU Load
mpstat -P ALL 10

# Check the summary of sync files
aws s3 ls --recursive s3://ray-batch-lab/zhy-upload/test_randfiles/ --region cn-northwest-1  | wc -l
16368
```

**Summary**
1. 3m7.929s (aws s3 cp) v.s 2m49.699s (aws s3 sync)
4. The total 16368 files at a rate of 16368 / 169sec = 96.85 files/sec (40961 MiB / 169sec = 242.37 MiB/sec) using 100 parallel thread + 10 Parallel Streams
4. When the file size larger than multipart_threshold=10 MiB, the multipart upload will be applied. 
5. Running multiple commands in parallel to maximize throughput



**Summary**:
1. Running the sync command to keep local and remote Amazon S3 locations synchronized over time. 
2. Synchronizing can be much faster than creating a new copy of the data in many cases.
3. Sync only handle the changed files
4. Sync will not remove files in desitnation bucket when source delete the files