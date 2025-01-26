# Pwnable.tw orw
## 需要掌握的知识点
无

## 代码分析
### main 方法

```cpp
int __cdecl main(int argc, const char **argv, const char **envp)
{
    orw_seccomp();
    printf("Give my your shellcode:");
    read(0, &shellcode, 0xC8u);
    ((void (*)(void))shellcode)();
    return 0;
}
```
很简单，似乎就是输入一段数据，然后将这段数据当做代码来执行

## 漏洞分析
### Stage 1 
分析代码，很简单哇。这连什么溢出都不需要了，直接在Read()函数调用时写下Shellcode就行。结果，哎..不行。

原来是在orw_seccomp()方法中调用了prctl()，这个函数对Execv()函数进行了屏蔽。

### Stage 2
既然Execv()不可以，那就另辟蹊径，这里直接就说了把，就是利用open, read函数进行读取flag 文件，write函数写到标准输出就行。

## 总结
- 经验
    - 有的时候execv()函数失败，查看是否使用了prctl()
- 知识
    - prctl()的认识
