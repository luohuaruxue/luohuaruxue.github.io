title: JVM 命令脚本
date: 2016-04-04 20:53:30

---

### 查看 Jvm  当前状态

* 查看 tomcat pid
```bash
ps -ef |grep tomcat
/JAVA_HOME/bin/jps
```

* 查看当前内存情况
```bash
jstat -gcutil pid > 
```

* 导出线程
```bash
jstack pid >
```

* 输出概要信息
```bash
jmap -heap pid >
```

* 导出 heap
```bash
jmap -dump:live,format=b,file=heap.bin $1
```

### 脚本

```bash
echo 'tomcat process:'$1
dirname=`date '+%Y%m%d%T'`
mkdir $dirname
chmod 777 $dirname
echo 'make directory:'$dirname
echo 'begin dump stack log..,log:'$dirname'/stack.log'
jstack $1 >$dirname'/stack.log'
echo 'begin dump stack log..,log:'$dirname'/heap.log'
jmap -heap $1 >$dirname'/heap.log'
su - tomcat -c "jmap -dump:live,format=b,file=${dirname}/heap.bin $1"
```

* [MAT分析](http://seanhe.iteye.com/blog/898277)

* 快速找到占用对象
```bash
 jmap -histo pid | sort -n -r -k 2 >
```

* [Java Bin 常用命令](http://nolinux.blog.51cto.com/4824967/1588716
http://blog.csdn.net/feihong247/article/details/7874063)
* [JVM 内存模型](http://blog.csdn.net/ithomer/article/details/6252552)
