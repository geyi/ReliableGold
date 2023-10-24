# jmap -heap 1
```
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.171-b11

using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 4294967296 (4096.0MB)
   NewSize                  = 1073741824 (1024.0MB)
   MaxNewSize               = 1073741824 (1024.0MB)
   OldSize                  = 3221225472 (3072.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 966393856 (921.625MB)
   used     = 559082632 (533.182746887207MB)
   free     = 407311224 (388.44225311279297MB)
   57.85246134677413% used
Eden Space:
   capacity = 859045888 (819.25MB)
   used     = 546122408 (520.8229141235352MB)
   free     = 312923480 (298.42708587646484MB)
   63.57313568795058% used
From Space:
   capacity = 107347968 (102.375MB)
   used     = 12960224 (12.359832763671875MB)
   free     = 94387744 (90.01516723632812MB)
   12.073096716651404% used
To Space:
   capacity = 107347968 (102.375MB)
   used     = 0 (0.0MB)
   free     = 107347968 (102.375MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 3221225472 (3072.0MB)
   used     = 761690744 (726.404899597168MB)
   free     = 2459534728 (2345.595100402832MB)
   23.645992825428646% used

59206 interned Strings occupying 6765192 bytes.
```

# YGC + FGC
```
2023-05-19T18:40:24.687+0800: 379299.130: [GC (Allocation Failure) 2023-05-19T18:40:24.687+0800: 379299.131: [ParNew
Desired survivor size 53673984 bytes, new threshold 15 (max 15)
- age   1:    3197440 bytes,    3197440 total
- age   2:     496744 bytes,    3694184 total
- age   3:     711760 bytes,    4405944 total
- age   4:     657792 bytes,    5063736 total
- age   5:     384432 bytes,    5448168 total
- age   6:     289136 bytes,    5737304 total
- age   7:     314672 bytes,    6051976 total
- age   8:     254016 bytes,    6305992 total
- age   9:     236144 bytes,    6542136 total
- age  10:     326232 bytes,    6868368 total
- age  11:     235128 bytes,    7103496 total
- age  12:     221696 bytes,    7325192 total
- age  13:     202352 bytes,    7527544 total
- age  14:     240328 bytes,    7767872 total
- age  15:     277400 bytes,    8045272 total
: 850228K->11765K(943744K), 0.0165180 secs] 3366786K->2528538K(4089472K), 0.0172823 secs] [Times: user=0.20 sys=0.00, real=0.02 secs]

2023-05-19T18:40:24.709+0800: 379299.152: [GC (CMS Initial Mark) [1 CMS-initial-mark: 2516773K(3145728K)] 2528609K(4089472K), 0.0074230 secs] [Times: user=0.03 sys=0.00, real=0.01 secs]
2023-05-19T18:40:24.717+0800: 379299.160: [CMS-concurrent-mark-start]
2023-05-19T18:40:26.548+0800: 379300.992: [CMS-concurrent-mark: 1.811/1.832 secs] [Times: user=7.20 sys=0.22, real=1.83 secs]
2023-05-19T18:40:26.549+0800: 379300.992: [CMS-concurrent-preclean-start]
2023-05-19T18:40:26.826+0800: 379301.269: [CMS-concurrent-preclean: 0.270/0.277 secs] [Times: user=0.32 sys=0.00, real=0.28 secs]
2023-05-19T18:40:26.826+0800: 379301.270: [CMS-concurrent-abortable-preclean-start] CMS: abort preclean due to time
2023-05-19T18:40:31.870+0800: 379306.313: [CMS-concurrent-abortable-preclean: 4.961/5.044 secs] [Times: user=6.29 sys=0.36, real=5.04 secs]
2023-05-19T18:40:31.877+0800: 379306.321: [GC (CMS Final Remark) [YG occupancy: 147614 K (943744 K)]
2023-05-19T18:40:31.878+0800: 379306.321: [Rescan (parallel) , 0.0564150 secs]
2023-05-19T18:40:31.934+0800: 379306.377: [weak refs processing, 0.9622571 secs]
2023-05-19T18:40:32.896+0800: 379307.340: [class unloading, 0.2960423 secs]
2023-05-19T18:40:33.192+0800: 379307.636: [scrub symbol table, 0.1784007 secs]
2023-05-19T18:40:33.371+0800: 379307.814: [scrub string table, 0.0076530 secs][1 CMS-remark: 2516773K(3145728K)] 2664387K(4089472K), 1.7770053 secs] [Times: user=1.82 sys=0.00, real=1.78 secs]
2023-05-19T18:40:33.655+0800: 379308.099: [CMS-concurrent-sweep-start]
2023-05-19T18:40:37.291+0800: 379311.734: [CMS-concurrent-sweep: 3.122/3.636 secs] [Times: user=5.36 sys=0.43, real=3.64 secs]
2023-05-19T18:40:37.291+0800: 379311.735: [CMS-concurrent-reset-start]
2023-05-19T18:40:37.302+0800: 379311.745: [CMS-concurrent-reset: 0.010/0.011 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]

2023-05-19T18:41:05.589+0800: 379340.032: [GC (Allocation Failure) 2023-05-19T18:41:05.589+0800: 379340.033: [ParNew
Desired survivor size 53673984 bytes, new threshold 15 (max 15)
- age   1:    4051120 bytes,    4051120 total
- age   2:     701392 bytes,    4752512 total
- age   3:     291944 bytes,    5044456 total
- age   4:     615040 bytes,    5659496 total
- age   5:     415904 bytes,    6075400 total
- age   6:     367992 bytes,    6443392 total
- age   7:     258472 bytes,    6701864 total
- age   8:     284448 bytes,    6986312 total
- age   9:     250712 bytes,    7237024 total
- age  10:     232664 bytes,    7469688 total
- age  11:     297736 bytes,    7767424 total
- age  12:     232176 bytes,    7999600 total
- age  13:     207200 bytes,    8206800 total
- age  14:     200368 bytes,    8407168 total
- age  15:     239264 bytes,    8646432 total
: 850677K->11438K(943744K), 0.1495685 secs] 1796186K->957217K(4089472K), 0.1502976 secs] [Times: user=1.05 sys=0.00, real=0.15 secs]
```


# FGC
```
2023-06-30T17:37:42.144+0800: 1886710.641: [GC (CMS Initial Mark) [1 CMS-initial-mark: 2516722K(3145728K)] 2530814K(4089472K), 0.0061590 secs] [Times: user=0.03 sys=0.01, real=0.00 secs]
2023-06-30T17:37:42.151+0800: 1886710.648: [CMS-concurrent-mark-start]
2023-06-30T17:37:45.283+0800: 1886713.780: [CMS-concurrent-mark: 3.132/3.132 secs] [Times: user=12.62 sys=0.00, real=3.14 secs]
2023-06-30T17:37:45.283+0800: 1886713.780: [CMS-concurrent-preclean-start]
2023-06-30T17:37:45.476+0800: 1886713.973: [CMS-concurrent-preclean: 0.180/0.192 secs] [Times: user=0.29 sys=0.00, real=0.19 secs]
2023-06-30T17:37:45.476+0800: 1886713.973: [CMS-concurrent-abortable-preclean-start] CMS: abort preclean due to time
2023-06-30T17:37:50.486+0800: 1886718.983: [CMS-concurrent-abortable-preclean: 4.891/5.010 secs] [Times: user=7.02 sys=0.00, real=5.01 secs]
2023-06-30T17:37:50.491+0800: 1886718.987: [GC (CMS Final Remark) [YG occupancy: 300877 K (943744 K)]
2023-06-30T17:37:50.491+0800: 1886718.988: [Rescan (parallel) , 0.0645744 secs]
2023-06-30T17:37:50.555+0800: 1886719.052: [weak refs processing, 0.5222228 secs]
2023-06-30T17:37:51.078+0800: 1886719.574: [class unloading, 0.2092175 secs]
2023-06-30T17:37:51.287+0800: 1886719.784: [scrub symbol table, 0.0482633 secs]
2023-06-30T17:37:51.335+0800: 1886719.832: [scrub string table, 0.0039043 secs][1 CMS-remark: 2516722K(3145728K)] 2817599K(4089472K), 1.0156883 secs] [Times: user=1.42 sys=0.00, real=1.02 secs]
2023-06-30T17:37:51.507+0800: 1886720.004: [CMS-concurrent-sweep-start]
2023-06-30T17:37:53.449+0800: 1886721.946: [CMS-concurrent-sweep: 1.918/1.942 secs] [Times: user=3.41 sys=0.11, real=1.94 secs]
2023-06-30T17:37:53.449+0800: 1886721.946: [CMS-concurrent-reset-start]
2023-06-30T17:37:53.456+0800: 1886721.953: [CMS-concurrent-reset: 0.007/0.007 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
```


# GC导致接口超时的日志
```
1074370.765: [GC (CMS Final Remark) [YG occupancy: 1161721 K (2831168 K)]
1074370.765: [Rescan (parallel) , 0.0282716 secs]
1074370.793: [weak refs processing, 3.9080203 secs]
1074374.701: [class unloading, 0.0895909 secs]
1074374.791: [scrub symbol table, 0.0193149 secs]
1074374.810: [scrub string table, 0.0073974 secs][1 CMS-remark: 4456982K(5242880K)] 5618704K(8074048K), 4.0599904 secs] [Times: user=3.06 sys=1.86, real=4.06 secs]

2020-07-06T13:41:27.510+0800: 98027.086: [GC (CMS Final Remark) [YG occupancy: 1143054 K (2831168 K)]
2020-07-06T13:41:27.511+0800: 98027.086: [Rescan (parallel) , 0.1128933 secs]
2020-07-06T13:41:27.624+0800: 98027.199: [weak refs processing
2020-07-06T13:41:27.624+0800: 98027.199: [SoftReference, 1487 refs, 0.0040026 secs]
2020-07-06T13:41:27.628+0800: 98027.203: [WeakReference, 15243 refs, 0.0061822 secs]
2020-07-06T13:41:27.634+0800: 98027.209: [FinalReference, 45882 refs, 0.3683047 secs]
2020-07-06T13:41:28.002+0800: 98027.578: [PhantomReference, 0 refs, 1 refs, 0.0032547 secs]
2020-07-06T13:41:28.005+0800: 98027.581: [JNI Weak Reference, 0.0000793 secs], 0.3819463 secs]
2020-07-06T13:41:28.005+0800: 98027.581: [class unloading, 0.1211467 secs]2020-07-06T13:41:28.127+0800: 98027.702: [scrub symbol table, 0.0214804 secs]
2020-07-06T13:41:28.148+0800: 98027.724: [scrub string table, 0.0116539 secs][1 CMS-remark: 4457263K(5242880K)] 5600318K(8074048K), 0.6552275 secs] [Times: user=11.70 sys=3.07, real=0.65 secs]
```



# 生产环境中价格服务的一次YGC日志
```
2023-07-01T15:45:06.534+0800: 1966189.986: [GC (Allocation Failure) 2023-07-01T15:45:09.132+0800: 1966192.584: [ParNew
Desired survivor size 53673984 bytes, new threshold 15 (max 15)
- age   1:    7400224 bytes,    7400224 total
- age   2:     496504 bytes,    7896728 total
- age   3:     716440 bytes,    8613168 total
- age   4:     390104 bytes,    9003272 total
- age   5:     281720 bytes,    9284992 total
- age   6:     473560 bytes,    9758552 total
- age   7:     541184 bytes,   10299736 total
- age   8:     319816 bytes,   10619552 total
- age   9:     368264 bytes,   10987816 total
- age  10:     414376 bytes,   11402192 total
- age  11:     302696 bytes,   11704888 total
- age  12:     407504 bytes,   12112392 total
- age  13:     283832 bytes,   12396224 total
- age  14:     297768 bytes,   12693992 total
- age  15:     339760 bytes,   13033752 total
: 852294K->16354K(943744K), 0.0159763 secs] 3232469K->2396780K(4089472K), 2.6147856 secs] [Times: user=0.13 sys=0.05, real=2.61 secs]
```
'real' time took more than 'usr' + 'sys' time. 说明应用程序因为缺少计算机资源而等待。


# GCLocker Initiated GC
```
2023-07-01T17:08:00.380+0800: 1971163.832: [GC (GCLocker Initiated GC) 2023-07-01T17:08:00.380+0800: 1971163.832: [ParNew
Desired survivor size 53673984 bytes, new threshold 15 (max 15)
- age   1:      14656 bytes,      14656 total
- age   2:    3728224 bytes,    3742880 total
- age   3:     732392 bytes,    4475272 total
- age   4:     525176 bytes,    5000448 total
- age   5:     377720 bytes,    5378168 total
- age   6:     661064 bytes,    6039232 total
- age   7:     399744 bytes,    6438976 total
- age   8:     398920 bytes,    6837896 total
- age   9:     311336 bytes,    7149232 total
- age  10:     384024 bytes,    7533256 total
- age  11:     352360 bytes,    7885616 total
- age  12:     294536 bytes,    8180152 total
- age  13:     304176 bytes,    8484328 total
- age  14:     292552 bytes,    8776880 total
- age  15:     277832 bytes,    9054712 total
: 15664K->13235K(943744K), 0.0122205 secs] 2467730K->2465587K(4089472K), 0.0125392 secs] [Times: user=0.15 sys=0.00, real=0.01 secs] 
```
当JNI关键区域被释放时，GC被触发。当本机代码使用JNI `GetPrimitiveArrayCritical()`或`GetStringCritical()`函数获取对Java堆中数组或字符串的引用时，就会出现JNI关键区域。该引用一直保持到本机代码调用相应的发布函数为止。从**获取**到**发布**之间的时间被称为JNI关键部分，在此期间，Java VM无法达到允许垃圾收集发生的状态。当任何线程处于JNI关键区域时，GC都会被阻塞。如果在这段时间内请求了GC，那么在所有线程从JNI关键区域退出后，将调用该GC。

## 触发GC的原因
- Allocation Failure：内存分配失败
- GCLocker Initiated GC
- Ergonomics：JVM自己进行自适应调整引发的GC
- Metadata GC Threshold：MetaSpace被占满