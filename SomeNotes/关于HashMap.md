## HashMap indexFor

```ruby
static int indexFor(int h, int length){
    return h & (length - 1);
}
```
1.当 length为2 的n 次方时， 上述操作就相当于对length的取模运算，而且速度比直接取模快的多   
2.为什么是2 的 n 次方，有另一个很重要的原因是为了均匀分布table数据和充分利用空间。当length等于2的n 次方时，不同的hash值发生的碰撞概率较小，使得数据的分布比较均匀，查询速度也较快。