# The aws s3 cli testing for Cross-Region transfer
## Test case 4: Periodically Synchronizing a Directory

### Configure the s3 cli
```bash
aws configure set default.s3.max_concurrent_requests 100
aws configure set default.s3.max_queue_size 10000
aws configure set default.s3.multipart_threshold 10MB
aws configure set default.s3.multipart_chunksize 5MB
```
https://docs.aws.amazon.com/cli/latest/topic/s3-config.html


## 1. create source documents
```bash
cd s3cli-testing

i=1;
while [[ $i -le 512 ]]; do
    num=$((8192/$i))
    [[ $num -ge 1 ]] || num=1
    mkdir -p randfiles/$i
    seq -w 1 $num | xargs -n1 -P 256 -I % dd if=/dev/urandom of=randfiles/$i/randfile_$i.% bs=512k count=$i;
    i=$(($i*2))
done

du -sm randfiles/
40961	randfiles/

find ./randfiles/ -type f | wc -l
16368
```

## 2. EC2 to S3 sync
1. Single Stream with multiple threads
```bash
cd randfiles/
time aws s3 sync --quiet . s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ --region ap-east-1
real	6m18.772s
user	13m34.046s
sys	2m19.694s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
100

# Check the summary of sync files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ --region ap-east-1  | wc -l
16368
```

2. Parallel Streams with multiple threads
```bash
# - xargs launches up to 16 parallel (‘-P16’) invocations of aws s3 cp. Only 10 are actually launched based on the output of the find.
i=1;
while [[ $i -le 512 ]]; do
    touch $i/*
    i=$(($i*2))
done
time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 sync --quiet {}/ s3://molex-mess-migration-hk/zhy-upload/test_randfiles/{}/ --region ap-east-1 )
real	1m56.399s
user	8m44.562s
sys	5m9.768s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
around 965, less than max_concurrent_requests * 10 = 1000

# CPU Load
mpstat -P ALL 10
02:43:02 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:43:12 AM  all    6.26    0.00    1.81    0.00    0.00    2.30    0.00    0.00    0.00   89.63

# Check the summary of sync files
aws s3 ls --recursive s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ --region ap-east-1  | wc -l
16368
```

**Summary**
1. 6m29.426s (aws s3 cp) v.s 6m18.772s (aws s3 sync)
2. 2m45.275s (aws s3 cp) v.s 1m56.399s (aws s3 sync)
3. The total 16368 files at a rate of 16368 / 378sec = 43.3 files/sec (40961 MiB / 378sec = 108.36 MiB/sec) using 100 parallel thread. 
4. The total 16368 files at a rate of 16368 / 116sec = 141.1 files/sec (40961 MiB / 116sec = 353.11 MiB/sec) using 100 parallel thread + 10 Parallel Streams
4. When the file size larger than multipart_threshold=10 MiB, the multipart upload will be applied. 
5. Running multiple commands in parallel to maximize throughput

## 3. Cros region S3 buckets sync
1. Single Stream with multiple threads
```bash
time aws s3 sync --quiet s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/ --source-region ap-east-1 --region ap-northeast-1
real	7m38.176s
user	4m24.859s
sys	0m22.213s

# capture the number of open connections
lsof -i tcp:443 | tail -n +2 | wc -l
100


# Check the summary of sync files
aws s3 ls --recursive s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/ --region ap-northeast-1 | wc -l
16368
```

2. Parallel Streams with multiple threads
```bash
# - xargs launches up to 16 parallel (‘-P16’) invocations of aws s3 cp. Only 10 are actually launched based on the output of the find.
# modify the data to allow re-sync
i=1;
while [[ $i -le 512 ]]; do
    touch $i/*
    i=$(($i*2))
done
find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 sync --quiet {}/ s3://molex-mess-migration-hk/zhy-upload/test_randfiles/{}/ --region ap-east-1

time ( find . -mindepth 1 -maxdepth 1 -type d -print0 | xargs -n1 -0 -P16 -I {} aws s3 sync --quiet s3://molex-mess-migration-hk/zhy-upload/test_randfiles/{}/ s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/{}/ --source-region ap-east-1 --region ap-northeast-1 )
real	1m5.924s
user	2m33.268s
sys	0m17.592s

# capture the number of open connections
around 217, less than max_concurrent_requests * 10 = 1000


# Check the summary of sync files
aws s3 ls --recursive s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/ -region ap-northeast-1 | wc -l
32736
```

1. 7m38.176s (aws s3 sync cross region bucket sync) v.s 6m18.772s (aws s3 sync local ec2 to S3)
2. Parallel Streams 1m5.924s (aws s3 sync cross region bucket sync) v.s Single Stream 7m38.176s (aws s3 sync cross region bucket sync)
3. The total 16368 files at a rate of 16368 / 458sec = 35.7 files/sec (40961 MiB / 458sec = 74.75 MiB/sec) using 100 parallel thread. 
4. The total 16368 files at a rate of 16368 / 66sec = 248 files/sec (40961 MiB / 66sec = 620.6 MiB/sec) using 100 parallel thread + 10 Parallel Streams
5. Running multiple commands in parallel to maximize throughput

## 4. delta sync
Only the touched and new files were transferred to Amazon S3.

```bash
touch 512/*
mkdir -p 10_more
seq -w 1 5 | xargs -n1 -P 5 -I % dd if=/dev/urandom of=10_more/10_more% bs=1024k count=10

find . –type f -mmin -10
./512/randfile_512.01
./512/randfile_512.02
./512/randfile_512.03
./512/randfile_512.04
./512/randfile_512.05
./512/randfile_512.06
./512/randfile_512.07
./512/randfile_512.08
./512/randfile_512.09
./512/randfile_512.10
./512/randfile_512.11
./512/randfile_512.12
./512/randfile_512.13
./512/randfile_512.14
./512/randfile_512.15
./512/randfile_512.16
./10_more
./10_more/10_more1
./10_more/10_more2
./10_more/10_more3
./10_more/10_more4
./10_more/10_more5

aws s3 sync . s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ --region ap-east-1
```

## 5. test delete
Source bucket files deletion will not sync to Destination bucket unless you use `--delete`
```bash
aws s3 rm --recursive s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/ --region ap-east-1
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.02
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.03
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.04
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.08
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.01
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.05
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.12
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.07
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.09
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.11
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.13
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.15
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.14
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.10
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.06
delete: s3://molex-mess-migration-hk/zhy-upload/test_randfiles/512/randfile_512.16

# will not remove in destination s3 bucket
aws s3 sync s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/ --source-region ap-east-1 --region ap-northeast-1
Nothing output

# will remove in destination s3 bucket with --delete
aws s3 sync s3://molex-mess-migration-hk/zhy-upload/test_randfiles/ s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/ --source-region ap-east-1 --region ap-northeast-1 --delete
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.01
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.09
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.11
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.16
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.12
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.05
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.04
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.14
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.15
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.07
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.08
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.10
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.03
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.02
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.13
delete: s3://molex-mess-migration-tokyo/crr-sync/test_randfiles/512/randfile_512.06
```


**Summary**:
1. Running the sync command to keep local and remote Amazon S3 locations synchronized over time. 
2. Synchronizing can be much faster than creating a new copy of the data in many cases.
3. Sync only handle the changed files
4. Sync will not remove files in desitnation bucket when source delete the files