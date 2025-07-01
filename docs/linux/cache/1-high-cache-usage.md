---
tags:
  - Linux
  - cache
  - high-cache
  - question
---
场景:
线上环境的server page cache使用到100%.  

解决:
监控server, 查看是哪个文件占用的cache 高, 以及此文件被哪个进程打开使用, 找到进程后, review code 查看其引起问题的原因


clear cache :(临时解决方案)
```shell
# 此出的 /var/log/lastlog 就是占用cache高的文件
dd if=/var/log/lastlog iflag=nocache count=0

```


monitor tool 
```shell
# https://github.com/Jason-fy-wang/MyShellLib/tree/master/shell/linux


# monitor_file.sh
### 查看文件被哪个进程修改

# scan_all_files.sh
### 扫描所有文件占用的cache情况

# get_process_file_cache.sh
### 查看某个 process 打开的文件的占用cache 情况
```

针对此情况可以用以下思路解决:
方法1:
1. 扫描所有文件, 找到占用cache高的文件; 并通过lsof查找占用此文件的process. 
2. 清除cache, 让server恢复正常
3. 监控此文件, 查看后续是哪个process操作此文件导致的cache高


[opensnoop](https://github.com/andreasgerstmayr/bpfcc/blob/master/tools/opensnoop.py)










