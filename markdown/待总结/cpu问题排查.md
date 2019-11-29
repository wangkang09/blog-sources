



SYS CPU高问题排查 @邹凯
1. top  --查出CPU高的进程PID
2. top -Hp pid  --根据进程查出占CPU高线程
3. jstack pid  > pid.log  -- 转储堆栈信息
4. 将第2步查到的线程ID 转换成16进制 查看pid.log 中线程信息，找出问题所在
5. 开发修复BUG
6. 验证







<https://blog.csdn.net/joeyon1985/article/details/39127087>  关于Linux系统指令 top 之 %si 占用高，分析实例一



<https://huataihuang.gitbooks.io/cloud-atlas/os/linux/kernel/tracing/diagnose_high_sys_cpu.html>  system use 高原因排查

<https://www.jianshu.com/p/bc9883864d2e> 同上





<https://my.oschina.net/wangen2009/blog/1542080>  java cpu us sys 高原因



<https://www.javazhiyin.com/30964.html>  java 分析 cpu 高原因，不错哦



<https://blog.csdn.net/mijichui2153/article/details/81325296> cup 利用率与平均负载关系





<https://cloud.tencent.com/developer/article/1165646>  多个业务优化完整流程



<http://www.linkedkeeper.com/172.html> rt 与 qps 关系



<https://kiswo.com/article/1028> Tomcat 压测记录