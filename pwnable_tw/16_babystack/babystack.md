BabyStack

也是一个很简单的题目，就写点概要吧。

--> 关键点在于strlen，因为是统计到'\x00'结尾，所以可以人为地进行控制结果
--> 以上漏洞的组合就是(value = strlen --> strcmp(value))，可以爆破，反复比较

脆弱代码:
s_len = strlen(&s);
if ( strncmp(&s, result, s_len) )
    return puts("Failed !");
