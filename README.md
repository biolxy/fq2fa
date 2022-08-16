# fq2fa 使用系统zlib和手动编译zlib速度差异很大！

## 1. 查看 fq2fa.c 文件

cat `fq2fa.c`

```c
#include <stdio.h>
#include <zlib.h>
#include "kseq.h"


KSEQ_INIT(gzFile, gzread)

int main(int argc, char *argv[])
{

        gzFile fp;
        gzFile fo;
        if (argc < 2 ){
            return -1;
        }
        if ( argc == 3 ){
            fo = gzopen (argv[2], "wb");
        }

        kseq_t *seq;
        int l;
        if (argc == 1){
            fprintf(stderr, "Usage: %s <in.fastq|in.fastq.gz> <out.fastq|out.fastq.gz>\n", argv[0]);
            return 1;
        }

        fp = gzopen(argv[1], "r");
        seq = kseq_init(fp); // 分配内存给seq
        while( (l = kseq_read(seq)) >= 0){ //读取数据到seq中
            gzprintf(fo, "%s", seq->name.s);
            gzprintf(fo, "%s", seq->seq.s);
        }

        kseq_destroy(seq); //释放内存
        gzclose(fp);
        if (argc == 3) gzclose(fo);
        return 0;
}
```


## 2. ubuntu20.04 系统安装 zlib 

```shell
apt-get install -y zlib1g zlib1g.dev zlib
```


## 3. 编译安装 zlib-1.2.11

```
cd /opt
wget https://github.com/madler/zlib/archive/refs/tags/v1.2.11.tar.gz
tar -zxvf v1.2.11.tar.gz
cd zlib-1.2.11
mkdir _build
./configure --prefix=/opt/zlib-1.2.11/_build --shared --static
make && make install
```

## 4. 编译链接 fq2fa.c


```
# 手动编译的zlib版本
gcc -o fq2fa_zlib-1.2.11 fq2fa.c /opt/zlib-1.2.11/_build/lib/libz.a -I/opt/zlib-1.2.11/_build/include
# apt-get 安装的 zlib版本（ cat /usr/lib/x86_64-linux-gnu/pkgconfig/zlib.pc 可以看到也是 1.2.11）
gcc -o fq2fa_zlib-ubuntu fq2fa.c -lz -Lzlib
```


## 5. 测试 fq2fa_zlib-1.2.11 和 fq2fa_zlib-ubuntu 版本各自的速度

### 生成测试文件

`raw.fq.gz` 太小，体现不出速度差异

```
Tfile=" "
for i in $(seq 1 100);do Tfile="$Tfile raw.fq.gz" ;done
cat ${Tfile} > in.fq.gz 
```

- 测试文件:
  - infile: `in.fq.gz`
- command:

```
/usr/bin/time -v ./fq2fa_zlib-ubuntu in.fq.gz out_fq2fa_zlib-ubuntu_.fa.gz 
/usr/bin/time -v ./fq2fa_zlib-1.2.11 in.fq.gz out_fq2fa_zlib-1.2.11_.fa.gz 
```

```
$ /usr/bin/time -v ./fq2fa_zlib-1.2.11 in.fq.gz out_fq2fa_zlib-1.2.11_.fa.gz 
	Command being timed: "./fq2fa_zlib-1.2.11 in.fq.gz out_fq2fa_zlib-1.2.11_.fa.gz"
	User time (seconds): 34.85
	System time (seconds): 0.10
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:34.97
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 1868
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 1
	Minor (reclaiming a frame) page faults: 258
	Voluntary context switches: 13
	Involuntary context switches: 78
	Swaps: 0
	File system inputs: 199576
	File system outputs: 92608
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```


```
$ /usr/bin/time -v ./fq2fa_zlib-ubuntu in.fq.gz out_fq2fa_zlib-ubuntu_.fa.gz 
	Command being timed: "./fq2fa_zlib-ubuntu in.fq.gz out_fq2fa_zlib-ubuntu_.fa.gz"
	User time (seconds): 22.03
	System time (seconds): 0.18
	Percent of CPU this job got: 99%
	Elapsed (wall clock) time (h:mm:ss or m:ss): 0:22.23
	Average shared text size (kbytes): 0
	Average unshared data size (kbytes): 0
	Average stack size (kbytes): 0
	Average total size (kbytes): 0
	Maximum resident set size (kbytes): 1876
	Average resident set size (kbytes): 0
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 162
	Voluntary context switches: 12
	Involuntary context switches: 31
	Swaps: 0
	File system inputs: 199392
	File system outputs: 92608
	Socket messages sent: 0
	Socket messages received: 0
	Signals delivered: 0
	Page size (bytes): 4096
	Exit status: 0
```

```
$ zcat in.fq.gz |wc -l 
4000000
```

## 6. 疑问？

- 为什么两个版本的程序速度差异这么大?


