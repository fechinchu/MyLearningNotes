# JVM详解13-JVM调优

# 1.OOM

如下是OOM产生的原因及其频率;

![image-20211112164721882](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112164721882.png)

![image-20211112164744763](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112164744763.png)

## 1.1.Java heap space

```java
@RequestMapping("/add")
    public void addObject(){
        System.err.println("add"+peopleSevice);
        ArrayList<People> people = new ArrayList<>();
        while (true){
            people.add(new People());
        }
    }
```

如上代码模拟线上OOM的情况

```shell
java -XX:+PrintGCDetails -XX:MetaspaceSize=64m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap/heapdump.hprof -XX:+PrintGCDateStamps -Xms
200M -Xmx200M -Xloggc:log/gc-oomHeap.log -jar demo-0.0.1-SNAPSHOT.jar 
```

发出请求,得如下响应:

![image-20211111222356783](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211111222356783.png)

这时候不妨查看日志,通过日志可以看出问题点

### 1.1.1.jvisualvm分析

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211111225227794.png" alt="image-20211111225227794" style="zoom:50%;" />

<img src="https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211111225339848.png" alt="image-20211111225339848" style="zoom:50%;" />

### 1.1.2.MAT分析

#### 1.1.2.1.Leak Suspect

在Leak Suspect中可以看到问题点

![image-20211112143037316](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112143037316.png)

#### 1.1.2.2.Thread Overview

![image-20211112145220650](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112145220650.png)

根据Leak Suspect获取到有问题的线程,在Thread Overview中可以看出问题点;

#### 1.1.2.3.Histogram

通过Histogram图中的incoming references也可以看出问题点

![image-20211112143526642](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112143526642.png)

### 1.1.3.GCEasy

https://gceasy.io/来分析GClog;

## 1.2.Metaspace

Java虚拟机规范对方法区的限制非常宽松,除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外,还可以选择不实现垃圾收集.垃圾收集行为在这个区域是比较少出现的,**其内存回收目标主要是针对常量池的回收和对类型的卸载**.当方法区无法满足内存分配需求时,将抛出OutOfMemoryError;

```java
@RequestMapping("/metaSpaceOom")
    public void metaSpaceOom(){
        ClassLoadingMXBean classLoadingMXBean = ManagementFactory.getClassLoadingMXBean();
        while (true){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(People.class);
            enhancer.setUseCache(false);
            //enhancer.setUseCache(true);
            enhancer.setCallback((MethodInterceptor) (o, method, objects, methodProxy) -> {
                System.out.println("我是加强类，输出print之前的加强方法");
                return methodProxy.invokeSuper(o,objects);
            });
            People people = (People)enhancer.create();
            people.print();
            System.out.println(people.getClass());
            System.out.println("totalClass:" + classLoadingMXBean.getTotalLoadedClassCount());
            System.out.println("activeClass:" + classLoadingMXBean.getLoadedClassCount());
            System.out.println("unloadedClass:" + classLoadingMXBean.getUnloadedClassCount());
        }
    }
```

```shell
java -XX:+PrintGCDetails -XX:MetaspaceSize=60m -XX:MaxMetaspaceSize=60m -Xss512K -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap/heapdumpMeta.hprof -XX:SurvivorRatio=8 -XX:+TraceClassLoading -XX:+TraceClassUnloading -XX:+PrintGCDateStamps -Xms60M -Xmx60M -Xloggc:log/gc-oomMeta.log demo-0.0.1-SNAPSHOT.jar
```

### 1.2.1.jvisualvm分析

![image-20211112165931772](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211112165931772.png)

### 1.2.2.MAT分析

在Histogram中group by package;可以看到用了大量的反射,创建了太多的类.导致Metaspace OOM;

![image-20211115094439530](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115094439530.png)

### 1.2.3.GCEasy

![image-20211115095413077](https://fechin-picgo.oss-cn-shanghai.aliyuncs.com/PicGo/image-20211115095413077.png)

### 1.2.4.原因以及解决

* 原因:
  * 运行期间生成了大量的代理类,导致方法区被撑爆,无法卸载;
  * 应用长时间运行,没有重启;
  * 元空间内存设置过小;
* 解决方法:
  * 检查是否永久代或者元空间设置的过小;
  * 检查代码中是否存在大量的反射操作;
  * dump之后通过MAT检查是否存在大量由于反射生成的代理类;

## 1.3.GC overhead limit exceeded

