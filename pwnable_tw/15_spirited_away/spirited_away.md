# pwnable tw spirited away

来源: https://www.taintedbits.com/2020/07/11/binary-exploitation-pwnable-tw-spirited-away/

## 程序分析
`main` 函数相当于就执行了一个 `survey` 函数, `survey` 函数是不断循环, 每一次都是用户输入姓名、年龄、评价等.

值得注意的是, 一共有几个字符串变量, 基本都在栈上, 只有 `name_ptr` 存储的是指向堆上的内存指针, 每次有一个申请内存的操作.

```cpp
int survey()
{
    char   respond[];     // [esp+0x10], [ebp-0xe8]
    int    name_len;      // [esp+0x48], [ebp-0xb0]
    int    reason_len;    // [esp+0x4c], [ebp-0xac]
    char   comment[0x3c]; // [esp+0x50], [ebp-0xa8]
    int    age;           // [esp+0xa0], [ebp-0x58]
    char   *name_ptr;     // [esp+0xa4], [ebp-0x54]
    char   reason[];      // [esp+0xa8], [ebp-0x50]

    name_len = 60;
    reason_len = 80;
LABEL_2:
    memset(&comment, 0, 0x50u);
    name_ptr = malloc(0x3Cu);
    // 输入
    printf("\nPlease enter your name: "); 
    read(0, name_ptr, name_len);
    printf("Please enter your age: "); 
    __isoc99_scanf("%d", &age);
    printf("Why did you came to see this movie? "); 
    read(0, &reason, reason_len);
    printf("Please enter your comment: "); 
    read(0, &comment, name_len);
    ++cnt;
    // 输出
    printf("Name: %s\n", name_ptr);
    printf("Age: %d\n", age);
    printf("Reason: %s\n", &reason);
    printf("Comment: %s\n\n", &comment);
    sprintf(&respond, "%d comment so far. We will review them as soon as we can", cnt);
    puts(&respond);
    if ( cnt > 199 ) {
        puts("200 comments is enough!");
        exit(0);
    }
    while ( 1 ) {
        printf("Would you like to leave another comment? <y/n>: ");
        read(0, &choice, 3);
        if ( choice == 'y' || choice == 'Y' )
        {
          free(buf);
          // 重新回到开头部分, 继续下一次的输入.
          goto LABEL_2;
        }
    }
    puts("Bye!");
    return 0;
}
```

## 漏洞点
1. 使用的是 `read` 函数，没有 `NULL` 截断，可以泄露栈信息。
```cpp
// ...
read(0, &reason, reason_len);
read(0, &comment, name_len);
// ...
printf("Reason: %s\n", &reason);
printf("Comment: %s\n\n", &comment);
```

2. `output` 长度为 0x38, 但是 `sprintf` 长度限制不到位. 如果变量 `cnt` 为三位数, 那么就溢出了. 最后一个字节 n 会正好改变了 `name_len` 变量. 这个变量在下一次重新输入环节里是决定着 `name_ptr` 指向的堆块内容 和 栈上的 `comment` 变量.
```cpp
char   respond[];     // [esp+0x10], [ebp-0xe8]
int    name_len;      // [esp+0x48], [ebp-0xb0]

read(0, name_ptr, name_len);
read(0, &comment, name_len);

sprintf(&respond, "%d comment so far. We will review them as soon as we can", cnt);
if (cnt > 200) { ... }
```

## 思路
1. 通过漏洞点一来泄露 `libc` 基址 和 栈地址.

2. 根据漏洞点二, 我们有溢出堆中内容 和 栈内容, 发现栈溢出不能溢出到返回地址那里, 所以比较棘手. 发现了比较重要的信息: 栈变量 `comment` 溢出可以改变 堆变量指针 `name_ptr`. 因此使用 `allocate to stack` 手法, 看之后操作.

3. 我们利用 `comment` 在栈中构造一个假的堆块,  然后将 `name_ptr` 也修改指向该堆块, 然后 `free` 掉, 这样 这个假的块 就在 fastbin 中了.

4. 接下来的重新输入, `name_ptr` 的 `malloc` 就会申请到这个假的堆块, 又因为可以溢出 `name_ptr` 指向内容, 所以可以修改返回地址. 之前说 `comment` 溢出无法修改返回地址, 而这个块起始位置比 `comment` 的位置要更靠近返回地址, 所以此时是可以修改返回地址的.

## 总结
1. allocate to stack
