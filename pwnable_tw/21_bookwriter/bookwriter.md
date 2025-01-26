# pwnable tw bookwriter

1. 泄露 Heap 基址
输入 name ，塞满，然后申请一个空间，然后打印
这样就会打印出 name 后面的内容，也就是申请的空间地址，从而得出Heap 基址

2. 泄露Libc 基址 --> unsorted
- 没有Free
---- 可以改变Top chunk 的size
使用Edit，塞满内容，此时更新strlen 更新size，直到00才停止，所以size 会变大，然后再次Edit 即可.

---- House of orange
可以修改Topchunk.size，这里有一些限制条件，最终改为0xfe1
此时申请一个大块，由于scanf 会直接申请大块，所以本题不用劳烦我们自己去申请了(TODO，这里不知道哪里调用scanf)
因为比Topchunk.size 大，所以会申请新的内存，并且将Top chunk 放入unsorted bin

- 得到unsorted bin 中内容
---- 这个就简单了，直接申请一块，那么就可以得到fd, bk；其中fd 会被我们的输入污染掉，但是bk 也可以

3. FSOP
- 把_IO_list_all 改写为 main_arena + 0x58 (unsorted bin addr)
---- page[8] 是不存在的，但是题目却可以申请，而这恰好改变的是size[0]
---- size[0] 改变完成后，我们就可以edit(0)，可以修改page[0] 后面的好长一段，足够我们去改变尚且在unsorted bin 中的那块内容
---- 很显然，修改那块的bk 为_IO_list_all - 0x10，然后两次申请，然后就改变了

- 修改small_bin[4] 内容 (size = 0x60)
---- 原因就是_IO_list_all->_chain，如果上面步骤一切正常，就是指向small_bin[4]

---- 这个地方又是一个很神奇的点。在我们修改Unsorted bin 唯一一块的bk 为 _IO_list_all-0x10 时，同时修改他的size = 0x61
---- (TODO，这里要调试一遍，为什么这一块就会被放在small_bin[4]上面呢??)

---- 修改small_bin[4] 内容，这地方就是相当于伪造一个vtable，这个之后具体看吧


_解释一下FSOP，原理就是修改一些东西之后，触发 _IO_flush_all_lockp，这个函数会对系统所有IO 进行fflush

对于本题
对所有IO 进行fflush，怎么找呢，首先就通过_IO_list_all 找到系统中的IO 链表。
然后chain 字段是每个IO 结构体都有的，可以理解为每个IO 结构体在链表中的下一个IO 结构体
所以会根据_IO_list_all->chain，然后开始fflush，然后再通过_IO_list_all->chain->chain，再开始fflush...

本题还有新的解法： https://0xffff.one/d/469
