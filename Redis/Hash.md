# Hash（ziplist、hashtable）
哈希类型的内部编码有两种：ziplist（压缩列表）、hashtable（哈希表）。只有当存储的数据量比较小的情况下，Redis 才使用压缩列表来实现字典类型。具体需要满足两个条件：
-  当哈希类型元素个数小于hash-max-ziplist-entries配置（默认512个）
-  所有值都小于hash-max-ziplist-value配置（默认64字节）

```
# Hashes are encoded using a memory efficient data structure when they have a
# small number of entries, and the biggest entry does not exceed a given
# threshold. These thresholds can be configured using the following directives.
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
```

## ziplist

```
非空 ziplist 示例图

area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                        ZIPLIST_ENTRY_TAIL

<uint32_t zlbytes> is an unsigned integer to hold the number of bytes that
the ziplist occupies, including the four bytes of the zlbytes field itself.
This value needs to be stored to be able to resize the entire structure
without the need to traverse it first.

<uint32_t zltail> is the offset to the last entry in the list. This allows
a pop operation on the far side of the list without the need for full
traversal.

<uint16_t zllen> is the number of entries. When there are more than
2^16-2 entries, this value is set to 2^16-1 and we need to traverse the
entire list to know how many items it holds.

<uint8_t zlend> is a special entry representing the end of the ziplist.
Is encoded as a single byte equal to 255. No other normal entry starts
with a byte set to the value of 255.
```

ziplist中的每个元素都包含了两部分前置信息。首先，存储了上一个条目的长度，以便能够从后向前遍历列表。其次，提供了条目的编码方式。它表示条目的类型，是整数还是字符串，并且对于字符串而言，还表示了字符串有效负载的长度。因此一个完整条目存储看起来像这样：
```
<prevlen> <encoding> <entry-data>
```
有时编码本身就代表着条目的内容，比如对于小整数来说。在这种情况下，<entry-data>部分是不存在的，我们可以只使用编码本身即可：
```
<prevlen> <encoding>
```

## 应用
- 保存用户信息
- 对hash中的元素进行数值计算