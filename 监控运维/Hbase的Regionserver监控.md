 ### 说明：
&emsp;&emsp;对应各个不同的环境，只要更改hbase的按照目录的路径就可以使用，同时加调度组件或者简单的corntab设置周期性执行。

#### 1 HBase regionserver 重启
```shell
#!/bin/bash
ProcNumber=`ps -ef |grep -w HRegionServer|grep -v grep|wc -l`
echo ${ProcNumber}
if [ ${ProcNumber} = 0 ]; then
   /usr/hdp/current/hbase-regionserver/bin/hbase-daemon.sh --config /usr/hdp/current/hbase-regionserver/conf start regionserver
   echo 'start hbase regionserver'
else
   echo 'hbase regionserver is living'
fi
```

#### 2.1 HBase regionserver GC 监控shell
```shell
result=$(cat /var/log/hbase/hbase-hbase-regionserver-hdp.log | grep 'JvmPauseMonitor')
if [ -z "$result" ]; then
    echo "result is empty"
fi
if [ -n "$result" ]; then
        local_host="`hostname --fqdn`"
    echo $result 2>/dev/null | python /home/ws_bgdata_devadmin/wpl_monitor/hbase_regionserver_gc.py $local_host 6
fi

```

#### 2.2 读取GC数据的python脚本
```python
#coding=utf-8
import json
import re
import sys
import time
from send_msg import SendMesg
from datetime import datetime,timedelta

def get_table_data(text,time_interval):
    out_info = {}
    now = time.strftime('%Y-%m-%d %H:%M:%S',time.localtime())
    end = datetime.fromtimestamp(time.mktime(time.strptime(now,'%Y-%m-%d %H:%M:%S')))
    arrInfo = ['='.join([i,j]) for i,j in zip(re.findall('\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3}',text),re.findall('approximately (.*?)ms',text))]
    for info in arrInfo:
        strInfo = info.split('=')
        log_time = strInfo[0].strip()[0:19]
        start = datetime.fromtimestamp(time.mktime(time.strptime(log_time, "%Y-%m-%d %H:%M:%S")))
        gc_time = strInfo[1]
        if int(gc_time) > 10000:
             if (end-start) <= timedelta(minutes=time_interval):
                 if gc_time not in out_info:
                    out_info[gc_time] = log_time
    return out_info

if __name__ == '__main__':
    host = sys.argv[1]
    time_interval = int(sys.argv[2]) # min
    text = sys.stdin.read()
    out_info = get_table_data(text,time_interval)
    if len(out_info) > 0:
        info_str = ''
        sender = SendMesg()
        for key in out_info.keys():
            info_str = info_str + "take:" + key+'ms, time:'+ out_info[key] +'\n'
        info_str = info_str + host
        print(info_str)
        print_info = sender.sendMsg("hadoop Hbase regionserver GC time > 1min", info_str)
        print(print_info)
```
