---
title: 线上系统频繁Full GC问题排查
date: 2019-09-20 15:02:03
categories:
- java
tags:
- JVM
- Full GC
- HashMap
- WeakHashMap
- dump
- MAT
---

# 现象描述
线上系统刚上线后运行正常，使用一段时间后出现频繁 Full GC 现象，系统出现卡顿，但是一直没有出现内存溢出，所以设置内存溢出后生成 dump 文件无效，重启后现象消失，运行一段时间后重复出现上述问题。

由于初期系统访问并发量并不是很大，而且改动上线比较频繁，所以问题并不明显，但是系统稳定后上线频率降低，运行比较长一段时间后就会出现上述问题。最后在线上系统频繁出现 Full GC 时通过 jmap 命令生成 java 堆 dump 文件，查看 java 堆内存中年轻代、年老代内存使用情况，最终确定了问题根源。

PS. 由于是事后复盘问题，所以线上监控系统监测到到的频繁 Full GC 现象没能截图保存下来，比较遗憾。

# 问题代码
```java
public class ParamsUtils implements Serializable {

    private static Map<String,JSONObject> JSON_OBJ_MAP = new HashMap<>();

    private static Object objForLock = new Object();

    private static int MAX_MAP_SIZE = 50000;

    /**
     * 获取已经JSONObject的对象
     * @param body Json字符串
     * @return json格式化后的对象
     */
    public static JSONObject getJson(String body){
        if (StringUtils.isBlank(body)) {
            return null;
        }
        String key = MD5Util.md5Hex(body);
        if (JSON_OBJ_MAP.containsKey(key)) {
            return JSON_OBJ_MAP.get(key);
        }
        JSONObject bodyObject = JSONObject.fromObject(body);
        if (!bodyObject.isNullObject() && !bodyObject.isEmpty() && !JSON_OBJ_MAP.containsKey(key)) {
            synchronized (objForLock) {
                if(!JSON_OBJ_MAP.containsKey(key)){
                    JSON_OBJ_MAP.put(key, bodyObject);
                }
            }
        }
        return bodyObject;
    }
}
```
上述代码的主要作用是解析网关接口的公共参数，将字符串参数转换为 JSON 对象，网关接口的格式如下：
```
// 关键敏感信息已脱敏
http://www.abc.com/client.action?functionId=test&body={"param1":"value1","param2":"value2","param3":"value3"}
```
ParamsUtils 类就是解析 URL 中 body 参数并转换为 JSON 对象的公共类。在转换过程中，主要经过以下几步：
- 解析参数对象转换为 JSON 对象；
- body 字符串进行 MD5 操作，生成 key；
- 判断 key 是否存在，不存在直接 put 进全局 JSON_OBJ_MAP 中。

JSON_OBJ_MAP 全局 Map 的主要作用应该是作为缓存使用，如果相同 body 参数已经解析过，不再重复解析，减少解析操作耗费的时间（可能想法是好的，但是实现的方式感觉并不优雅，后面我们再分析）。

# 复盘步骤 & 环境

为了更直白的说明问题，我在本地电脑准备了测试环境，通过一些特殊条件跟快速复现出问题现象，来说明问题：
- 本地通过 main 方法拼接字 body 参数符串，而且每次 body 参数内容都不相同，这样 JSON_OBJ_MAP 缓存失效，每次要放入内容都不同，会持续进行 put 操作；
- 本地通过 main 方法方式模拟大量请求解析参数过程，线上流量比较小，系统运行很长时间才会出现 Full GC，而且几乎不出现内存溢出情况，通过 mian 方法循环方式，可以快速模拟出现 Full GC 情况，而且一段时间后还会出现内存溢出；
- 本地通过 main 方法中创建的 body 字符串，通过某个参数使用长文本字符串，创建大字符串对象，快速消耗内存；
- 调整本地内存大小，线上服务器内存比较大，所以才会在运行一段时间后出现问题，测试环境可以减小 jvm 内存设置，使问题快速出现，而且还可以设置一些 jvm 垃圾回收参数，分析系统运行，直至出现内存溢出过程中内存使用情况。

```java
public static void main(String[] args) throws Exception {
    for (int i = 0; i < 5000000; i++) {
        String a = "{\"pin\":\"xxxxxxxx\",\"content\":\"测试数据测试数据\",\"index\":%s}";
        ParamsUtils.getJson(String.format(a, i));
    }
}
```

```
// 内存设置
-Xms500m -Xmx500m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump/java_pid.hprof -XX:+PrintGCDetails
```

# java 堆 dump 文件分析
<img src="http://ww1.sinaimg.cn/large/7dad8649ly1g75yodbmz6j21je0na78g.jpg" width = "600" alt="dump文件"/>

<img src="http://ww1.sinaimg.cn/large/7dad8649ly1g75yq8k07gj214c10wqpk.jpg" width = "600" alt="dump文件"/>

通过 MAT 工具查看 dump.hprof 文件，问题还是比较明显的（现在想不明白为什么当时定位这个问题花了好长时间，可能是“事后诸葛亮”，现在逆向反推感觉好简单^_^）。

主要的问题就是 JSON_OBJ_MAP 中持有大量对象不会释放，而且从系统运行角度而言，JSON_OBJ_MAP 只会一直往里 put 内容，不会进行移除，所以系统运行越久，JSON_OBJ_MAP 中存储的内容就越多，知道系统重启后，JSON_OBJ_MAP 中内容会被清空。

# 第一版代码修改
```java
public class ParamsUtils implements Serializable {

    private static Map<String,JSONObject> JSON_OBJ_MAP = new HashMap<>();

    private static Object objForLock = new Object();

    private static int MAX_MAP_SIZE = 50000;

    /**
     * 获取已经JSONObject的对象
     * @param body Json字符串
     * @return json格式化后的对象
     */
    public static JSONObject getJson(String body){
        if (StringUtils.isBlank(body)) {
            return null;
        }
        String key = MD5Util.md5Hex(body);
        if (JSON_OBJ_MAP.containsKey(key)) {
            return JSON_OBJ_MAP.get(key);
        }
        JSONObject bodyObject = JSONObject.fromObject(body);
        if (!bodyObject.isNullObject() && !bodyObject.isEmpty() && !JSON_OBJ_MAP.containsKey(key)) {
            synchronized (objForLock) {
                //释放
                if(JSON_OBJ_MAP.size() > MAX_MAP_SIZE) {
                    JSON_OBJ_MAP = new HashMap<>();
                }
                if(!JSON_OBJ_MAP.containsKey(key)){
                    JSON_OBJ_MAP.put(key, bodyObject);
                }
            }
        }
        return bodyObject;
    }
}
```
第一版修改后加入条件判断，如果 JSON_OBJ_MAP 中存储内容 > 50000，重新初始化，释放 Map 中内容；
```java
//释放
if(JSON_OBJ_MAP.size() > MAX_MAP_SIZE) {
    JSON_OBJ_MAP = new HashMap<>();
    // JSON_OBJ_MAP.clear(); //也可以通过调用 clear() 方法的方式。
}
```
其实这种方式上线后系统性能有所改观，但是还是没能避免前面的问题（当然这还是有一种事后分析的思路，因为最初修改后也感觉没问题）。

在 JSON_OBJ_MAP 中元素个数还是会从 0 增长到 5W，然后释放掉，再重复上述过程；其实对于 body 参数长度，根据不同的 URL 请求长度都是不固定的，如果某段时间 body 体比较大，也会占用比较多内存，而且当 JSON_OBJ_MAP 中元素个数接近 5W 而没有达到 5W 的时候还是会触发频繁的 Full GC。

我个人理解对于 5W 个元素这个限制，最大的作用是能够保证线上代码即使运行足够长时间也大概率不会出现内存溢出（只是大概率，如果某些 body 比较大，可能还没 5W 就直接内存溢出了），如果没有 5W 元素个数限制，线上代码运行足够长时间后理论上一定会出现内存溢出，我们线上的系统之所以没出现，可能是因为连续运行时间不足够长，还没到内存溢出条件就上线或重启等操作释放掉内存了。

# 第二版代码修改
```java
public class ParamsUtils implements Serializable {

    private static Map<String,JSONObject> JSON_OBJ_MAP = new WeakHashMap<>();

    private static Object objForLock = new Object();

    /**
     * 获取已经JSONObject的对象
     * @param body Json字符串
     * @return json格式化后的对象
     */
    public static JSONObject getJson(String body){
        if (StringUtils.isBlank(body)) {
            return null;
        }
        String key = MD5Util.md5Hex(body);
        if (JSON_OBJ_MAP.containsKey(key)) {
            return JSON_OBJ_MAP.get(key);
        }
        JSONObject bodyObject = JSONObject.fromObject(body);
        if (!bodyObject.isNullObject() && !bodyObject.isEmpty() && !JSON_OBJ_MAP.containsKey(key)) {
            synchronized (objForLock) {
                if(!JSON_OBJ_MAP.containsKey(key)){
                    JSON_OBJ_MAP.put(key, bodyObject);
                }
            }
        }
        return bodyObject;
    }
}
```
通过使用 WeakHashMap() ，不在使用强引用的HashMap()，WeakHashMap() 由于是弱引用，put 进去的元素也是可以被垃圾回收掉的，一般 Key 在 Young GC的时候就可以被回收掉，一般不会进入老年代出发 Full GC，可以解决这个问题，其实如果使用 WeakHashMap() 存储结构，5W 个元素的限制也可以去掉，不会有问题。

不过这里可能会有一个问题就是，JSON_OBJ_MAP 其实是作为一个本地缓存使用，如果使用 WeakHashMap() 存储结构，里面元素被快速垃圾回收掉，下次同样一个 body 参数体还需要进行解析，这样就降低了 JSON_OBJ_MAP 作为一个本地缓存的价值。

# 代码结构整理

# 题外话
除去前面提到的 Full GC 问题，单独看这段代码的实现，还是有值得思考的地方的，这是一个网关请求参数的公共解析类，线上系统流量进来之后，请求参数首先要这个类处理，这个类中对于全局变量 JSON_OBJ_MAP 的写操作是同步的，也就是如果有写操作，每次只有一个线程可以拿到锁，其它线程被阻塞，需要排队执等待。

如果是在高并发情况下，系统刚刚上线的时候，JSON_OBJ_MAP 为空，会触发大量的写操作，而排队写操作会导致每个请求进来都要排队进行参数解析，等待解析结果再执行业务逻辑，基本上可能把 HTTP 异步请求给转换成同步操作了，使用 JSON_OBJ_MAP 缓存解析结果可能在下次请求会节省一点时间，但是相比较同步阻塞的时间，可能有点得不偿失，而且目前业界开源的一些 JSON 解析工具，性能都是非常不错的，其实完全没有必要进行本地缓存，这个类完全可以改造成只是进行公共参数解析，不进行本地缓存。

# 工具命令
```
// 查看Java进程
jps -l 
```

## 如何查看 Java 进程
```
// Linux
ps -ef | grep java
jps -l （显示java进程的Id和软件名称）
jps -lmv（显示java进程的Id和软件名称；显示启动main输入参数；虚拟机参数）

// Windows
jps
jps -l（显示java进程的Id和软件路径及名称）
```

## Mac 系统 VisualVM 启动
```bash
1、/usr/libexec/java_home
2、cd /Library/Java/JavaVirtualMachines/jdk1.8.0_211.jdk/Contents/Home/bin
3、jvisualvm

就会弹出java visulvm了
```

## JDK 8 jmap 命令失败问题

```
// jmap 命令
jmap -F -dump:format=b,file=/data/dump/apidump.hprof 55967

// 异常信息
Attaching to process ID 55967, please wait...
Exception in thread "main" java.lang.reflect.InvocationTargetException
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at sun.tools.jmap.JMap.runTool(JMap.java:197)
	at sun.tools.jmap.JMap.main(JMap.java:128)
Caused by: sun.jvm.hotspot.runtime.VMVersionMismatchException: Supported versions are 24.80-b11. Target VM is 25.152-b26
	at sun.jvm.hotspot.runtime.VM.checkVMVersion(VM.java:234)
	at sun.jvm.hotspot.runtime.VM.<init>(VM.java:297)
	at sun.jvm.hotspot.runtime.VM.initialize(VM.java:368)
	at sun.jvm.hotspot.bugspot.BugSpotAgent.setupVM(BugSpotAgent.java:598)
	at sun.jvm.hotspot.bugspot.BugSpotAgent.go(BugSpotAgent.java:493)
	at sun.jvm.hotspot.bugspot.BugSpotAgent.attach(BugSpotAgent.java:331)
	at sun.jvm.hotspot.tools.Tool.start(Tool.java:163)
	at sun.jvm.hotspot.tools.HeapDumper.main(HeapDumper.java:77)
	... 6 more

```

有资料说是执行命令用户权限不够，我切换到 root 用户依然不好用；还有资料说是 JDK 8 中的 BUG，环境变量切换到 JDK 7 再执行 jmap 命令果然OK了，也许真的是 JDK 8 中引入的问题。

切换 JDK 环境变量命令：

```bash
export JAVA_HOME='/Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home'
```

[Mac 使用 jinfo 出现：Can't attach to the process. Could be caused by an incorrect pid or lack of privileg](https://blog.csdn.net/Dongguabai/article/details/88736589)