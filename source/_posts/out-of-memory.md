title: OutOfMemory 怎么办
date: 2016-04-05 20:42:48
---

### 背景：

线上系统内存泄露挂掉.

### 分析步骤：

由于内存泄露没有响应，已经无法强制导出内存，那么只能从日志和机器的监控图入手。 
从监控的机器数据图中可以看到，在内存泄露时，机器 CPU 已经跑高至 700+% 。
![top](http://ww4.sinaimg.cn/large/7317a86agw1f2m47gr0irj20tv0f9n42.jpg)

3. 日志方面，使用
```bash
cat worker.log |  grep "exception" > error.txt
```

4. 发现为 SwitchDispatchManager 线程抛出的无数的异常
```java
java.util.ConcurrentModificationException
        at java.util.HashMap$HashIterator.nextEntry(HashMap.java:793)
        at java.util.HashMap$KeyIterator.next(HashMap.java:828)
```

5. 查看代码发现
```java
private static Set<String> hasMsgSwitchIp = new HashSet<String>;
public void addMsg(String ip){
	...
	hasMsgSwitchIp.add(ip);

}
public void run() {
    while (true) {
        try {
            if (!hasMsgSwitchIp.isEmpty()){
                Iterator<String> ite = hasMsgSwitchIp.iterator();
                while(ite.hasNext()){
                    String ip = ite.next();
                    Queue<SwitchMessage> userObjs = msgMap.get(ip);
                    if (userObjs.isEmpty()) {
                        synchronized(userObjs){
                            if (userObjs.isEmpty()){
                                ite.remove();
                                continue;
                            }
                        }
                    }
                 ...
            }else{
                sleep(500);
            }
        }catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }
}
```

6. HashSet 并非线程安全，而addMsg()和 run()是不同线程操作，于是导致抛出异常,线程一直未能休眠，于是导致CPU异常跑高，系统跑挂

### 解决办法 
将 `HashSet` 改为 `Sets.newConcurrentHashSet()`;
