# Kafka如何高效读取指定offset的消息

Segment文件由两部分组成，分别为.index和.log，分别表示segment索引文件和数据文件。这两个文件的命名规则为：partition全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值，数值大小为64位，20位数字字符长度，没有数字使用0填充

.index索引文件存储大量的元数据，.log文件存储大量消息，索引文件中的元数据指向对应数据文件中message的物理偏移地址。其中以“.index”索引文件中的元数据[3, 348]为例，在“.log”数据文件表示第3个消息，即在全局partition中表示170410+3=170413个消息，该消息的物理偏移地址为348。

![20170506094723439](.\image\20170506094723439.png)

如何从partition中通过offset查找message读取offset= 170418，首先查找segment文件，其中00000000000000000000.index为最开始的文件，第二个文件为00000000000000170410.index（起始偏移为170410+1=170411），而第三个文件为00000000000000239430.index（起始偏移为239430+1=239431），所以这个offset=170418就落到了第二个文件之中。其他后续文件可以依次类推，以其实偏移量命名并排列这些文件，然后根据二分查找法就可以快速定位到具体文件位置。其次根据00000000000000170410.index文件中的[8,1325]定位到00000000000000170410.log文件中的1325的位置进行读取。

那么怎么知道何时读完本条消息，否则就读到下一条消息的内容了这个就需要联系到消息的物理结构了，消息都具有固定的物理结构，包括：offset（8 Bytes）、消息体的大小（4 Bytes）、crc32（4 Bytes）、magic（1 Byte）、attributes（1 Byte）、key length（4 Bytes）、key（K Bytes）、payload(N Bytes)等等字段，可以确定一条消息的大小，即读取到哪里截止。



| **关键字**            | **解释说明**                                                 |
| --------------------- | ------------------------------------------------------------ |
| 8 byte offset         | 在parition(分区)内的每条消息都有一个有序的id号，这个id号被称为偏移(offset),它可以唯一确定每条消息在parition(分区)内的位置。即offset表示partiion的第多少message |
| 4 byte message   size | message大小                                                  |
| 4 byte CRC32          | 用crc32校验message                                           |
| 1 byte “magic"        | 表示本次发布Kafka服务程序协议版本号                          |
| 1 byte “attributes"   | 表示为独立版本、或标识压缩类型、或编码类型。                 |
| 4 byte key   length   | 表示key的长度,当key为-1时，K byte key字段不填                |
| K byte key            | 可选                                                         |
| value bytes   payload | 表示实际消息数据。                                           |

