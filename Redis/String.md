Redis中的每个对象都由一个redisObject结构表示，该结构中与保存数据有关的三个属性分别是type、 encoding和ptr。
```c
// server.h
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    // 指向底层数据结构的指针
    void *ptr;
} robj;
```

type属性表示value的类型（相关命令：`type`）  
encoding属性表示value的编码方式（相关命令：`object encoding`）

Redis存储的数据是**二进制安全的**，它不会对存入的数据做任何处理或假设其为特定的数据类型，它能够接受和存储包含任何字节的数据。例如在存储一个“中”字时，使用UTF-8编码，会占3个字节，使用GBK编码，则会占2个字节。对于图像、音频、视频等文件，Redis同样将其视为二进制位组成的字节流。因此为了确保数据的一致性，连接同一个Redis服务的客户端应该使用相同的编码方式进行读写操作。

# String（int、embstr、raw）
- 字符串
- 整数
- bitmap

Redis会根据当前值的类型和长度来决定使用哪种编码方式。

Redis中的字符串分为两种存储方式，分别是embstr和raw，当字符串长度小于等于44的时候，Redis使用embstr来存储字符串，而当字符串长度超过44的时候，就需要用raw来存储。

embstr编码是专门用于保存短字符串的一种优化编码方式，相对于raw，embstr只会调用一次内存分配函数来分配一块连续的内存空间用于保存redisObject和SDS（将字符串直接嵌入在声明的数据结构中）。而raw编码则会调用两次内存分配函数分别分配两块空间来保存redisObject和SDS。

Redis并未直接使用C字符串，而是以struct的形式定义了一个SDS（Simple Dynamic String）类型。当Redis需要一个可以被修改的字符串时，就会使用SDS来表示。在Redis数据库里，包含字符串值的键值对都是由SDS实现的。SDS的结构如下：
```c
// sds.h
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4

// sds.c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}
```
- len：记录当前已使用的字节数（不包括 ‘\0’），获取 SDS 长度的时间复杂度为 O(1)
- alloc：记录当前字节数组总共分配的字节数量（不包括 ‘\0’）
- flags：低三位标记当前字节数组的属性，是 sdshdr5、sdshdr8 还是 sdshdr16 等
- buf：字节数组，用于保存字符串，包括结尾空白字符 ‘\0’

**Redis使用SDS的原因：**
- 获取字符串长度的时间复杂度为O(1)
- 杜绝缓冲区溢出
- 减少修改字符串时带来的内存重分配次数
- 惰性空间释放
- 二进制安全

## bitmap（raw）
bitmap在实际生产过程中的一次应用，每日设备推送开关状态统计，如下：
```
二进制从左往右的第一位表示id（自增主键）为1的设备，以此类推
A表示2020-1-1号的设备推送开关状态（0 0 1 1 1 1 0 0），B表示2020-1-2号的设备推送开关状态（1 1 1 1 0 0 1 0）
0 关
1 开
2020-1-1 4关4开
2020-1-2 3关5开
2号相比1号推送开关新增关闭的数量2
2号相比1号推送开关新增打开的数量3
C = A ^ B （1 1 0 0 1 1 1 0） 结果中1的数量表示开关发生变化的数量5
C & A （0 0 0 0 1 1 0 0） 结果中1的数量表示2号相比1号推送开关新关闭的数量2
C & B （1 1 0 0 0 0 1 0） 结果中1的数量表示2号相比1号推送开关新打开的数量3

8 * 1024 * 1024 = 8388608

1MB内存可以表示8388608个设备推送开关情况
```

类似的应用还有
- 每日所有用户登录状态统计
- 一个用户一年的登录状态统计