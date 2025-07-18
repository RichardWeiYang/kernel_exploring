虽然系统中引入了maple tree，但是我们看到还是有很多地方在使用xarray，尤其是address space。现在我们来比较一下，这两种数据结构。看看同样是存储数据，这两种数据结构是怎么处理的。

这里采用的方法是通过xa_dump()/mt_dump()实际观察数据结构中的变化。

# 存储值

## 单个值

我们同样在0x123这个index写入value 6，来看看这两个数据结构是怎么保存这个值的。

xarray代码

```
	xa_store(&my_array, 0x123, xa_mk_value(6), GFP_NOWAIT | __GFP_DIRECT_RECLAIM);
	xa_dump(&my_array);
	xa_destroy(&my_array);
```

maple tree代码

```
	mas_set(&mas_writer, 0x123);
	mas_store_gfp(&mas_writer, xa_mk_value(6), GFP_KERNEL);
	mt_dump(&tree, mt_dump_dec);
```

dump 输出

```
xarray:

xarray: 0x5d98839f3a00x head 0x60c000000102x flags 0 marks 0 0 0
0-511: node 0x60c000000100x max 0 parent (nil)x shift 6 count 1 values 0 array 0x5d98839f3a00x list 0x60c000000118x 0x60c000000118x marks 0 0 0
  256-319: node 0x60c0000001c0x offset 4 parent 0x60c000000100x shift 3 count 1 values 0 array 0x5d98839f3a00x list 0x60c0000001d8x 0x60c0000001d8x marks 0 0 0
    288-295: node 0x60c000000280x offset 4 parent 0x60c0000001c0x shift 0 count 1 values 1 array 0x5d98839f3a00x list 0x60c000000298x 0x60c000000298x marks 0 0 0
      291: value 6 (0x6) [0xdx]

maple tree:

maple_tree(0x7fffe1680eb0) flags 5, height 1 root 0x61500000010e
0-18446744073709551615: node 0x615000000100 depth 0 type 1 parent 0x7fffe1680eb1 contents: (nil) 290 0xd 291 (nil) 18446744073709551615 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 0x2
  0-290: (nil)
  291: value 6 (0x6) [0xd]
  292-18446744073709551615: (nil)
```

可以看到maple tree只用了一个node，而xarray用了三个node。可以说，此时maple tree完胜。

当然，如果我们存储一个较小的index，（8以内）。因为被一个xa_node覆盖，所以在xarray中也是可以用一个node表达。但是这种情况在实际使用中太少了。

## 两个有一定间隔的值

在上面的基础上，我们再看看此时再一个有一定距离的index上存储一个值会如何？

xarray代码

```
	xa_store(&my_array, 5, xa_mk_value(5), GFP_NOWAIT | __GFP_DIRECT_RECLAIM);
	xa_store(&my_array, 0x123, xa_mk_value(6), GFP_NOWAIT | __GFP_DIRECT_RECLAIM);
	xa_dump(&my_array);
	xa_destroy(&my_array);
```

maple tree代码

```
	mas_set(&mas_writer, 5);
	mas_store_gfp(&mas_writer, xa_mk_value(5), GFP_KERNEL);
	mas_set(&mas_writer, 0x123);
	mas_store_gfp(&mas_writer, xa_mk_value(6), GFP_KERNEL);
	mt_dump(&tree, mt_dump_dec);
```

dump 输出

```
xarray:

xarray: 0x5f23e8340280x head 0x60c000000282x flags 0 marks 0 0 0
  0-511: node 0x60c000000280x max 0 parent (nil)x shift 6 count 2 values 0 array 0x5f23e8340280x list 0x60c000000298x 0x60c000000298x marks 0 0 0
    0-63: node 0x60c0000001c0x offset 0 parent 0x60c000000280x shift 3 count 1 values 0 array 0x5f23e8340280x list 0x60c0000001d8x 0x60c0000001d8x marks 0 0 0
      0-7: node 0x60c000000100x offset 0 parent 0x60c0000001c0x shift 0 count 1 values 1 array 0x5f23e8340280x list 0x60c000000118x 0x60c000000118x marks 0 0 0
        5: value 5 (0x5) [0xbx]
    256-319: node 0x60c000000340x offset 4 parent 0x60c000000280x shift 3 count 1 values 0 array 0x5f23e8340280x list 0x60c000000358x 0x60c000000358x marks 0 0 0
      288-295: node 0x60c000000400x offset 4 parent 0x60c000000340x shift 0 count 1 values 1 array 0x5f23e8340280x list 0x60c000000418x 0x60c000000418x marks 0 0 0
        291: value 6 (0x6) [0xdx]

maple tree:

maple_tree(0x7ffd88e59c40) flags 5, height 1 root 0x61500000010e
0-18446744073709551615: node 0x615000000100 depth 0 type 1 parent 0x7ffd88e59c41 contents: (nil) 4 0xb 5 (nil) 290 0xd 291 (nil) 18446744073709551615 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 0x4
  0-4: (nil)
  5: value 5 (0x5) [0xb]
  6-290: (nil)
  291: value 6 (0x6) [0xd]
  292-18446744073709551615: (nil)
```

和上面的情况类似，对于xarray，index的位置是固定的，所以为了保存到index 5，xarray必须把和5有关的那些index位置也分配出来。而mapple tree则不需要，直接挤在一个节点就可以了。

# 存储范围

接下来我们看看存储一个范围的情况。

xarray 代码

```
	xa_store_range(&my_array, 0x123, 0x123 + 2, xa_mk_value(6), GFP_NOWAIT | __GFP_DIRECT_RECLAIM);
	xa_dump(&my_array);
	xa_destroy(&my_array);
```

maple tree 代码

```
```

dump 输出

```
xarray:

xarray: 0x5e4359f75400x head 0x60c000000102x flags 0 marks 0 0 0
0-511: node 0x60c000000100x max 0 parent (nil)x shift 6 count 1 values 0 array 0x5e4359f75400x list 0x60c000000118x 0x60c000000118x marks 0 0 0
  256-319: node 0x60c0000001c0x offset 4 parent 0x60c000000100x shift 3 count 1 values 0 array 0x5e4359f75400x list 0x60c0000001d8x 0x60c0000001d8x marks 0 0 0
    288-295: node 0x60c000000280x offset 4 parent 0x60c0000001c0x shift 0 count 3 values 3 array 0x5e4359f75400x list 0x60c000000298x 0x60c000000298x marks 0 0 0
      291: value 6 (0x6) [0xdx]
      292: sibling (slot 3)
      293: sibling (slot 3)

maple tree:

maple_tree(0x7ffd66fac520) flags 5, height 1 root 0x61500000010e
0-18446744073709551615: node 0x615000000100 depth 0 type 1 parent 0x7ffd66fac521 contents: (nil) 290 0xd 293 (nil) 18446744073709551615 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 0x2
  0-290: (nil)
  291-293: value 6 (0x6) [0xd]
  294-18446744073709551615: (nil)
```

同样是在一段区域内存储，maple tree只占用一个slot，而xarray则是要占用很多（占用的数量和区域大小一致）。

并且以此类推，如果在xarray中，我们扩大一下存储的区域。

xarray 代码

```
	xa_store_range(&my_array, 0x123, 0x123 + 5, xa_mk_value(6), GFP_NOWAIT | __GFP_DIRECT_RECLAIM);
	xa_dump(&my_array);
	xa_destroy(&my_array);
```

此时 dump 出来的结果就是

```
xarray:

xarray: 0x56512f750300x head 0x60c000000102x flags 0 marks 0 0 0
  0-511: node 0x60c000000100x max 0 parent (nil)x shift 6 count 1 values 0 array 0x56512f750300x list 0x60c000000118x 0x60c000000118x marks 0 0 0
    256-319: node 0x60c0000001c0x offset 4 parent 0x60c000000100x shift 3 count 2 values 0 array 0x56512f750300x list 0x60c0000001d8x 0x60c0000001d8x marks 0 0 0
      288-295: node 0x60c000000340x offset 4 parent 0x60c0000001c0x shift 0 count 5 values 5 array 0x56512f750300x list 0x60c000000358x 0x60c000000358x marks 0 0 0
        291: value 6 (0x6) [0xdx]
        292: sibling (slot 3)
        293: sibling (slot 3)
        294: sibling (slot 3)
        295: sibling (slot 3)
      296-303: node 0x60c000000280x offset 5 parent 0x60c0000001c0x shift 0 count 1 values 1 array 0x56512f750300x list 0x60c000000298x 0x60c000000298x marks 0 0 0
        296: value 6 (0x6) [0xdx]
```

而同样是操作区域[0x123, 0x123 + 5], maple tree则还是用一个slot来代表。

maple tree 代码

```
	mas_set_range(&mas_writer, 0x123, 0x123 + 5);
	mas_store_gfp(&mas_writer, xa_mk_value(6), GFP_KERNEL);
	mt_dump(&tree, mt_dump_dec);
```

dump 输出

```
maple tree

maple_tree(0x7ffd796e3e70) flags 5, height 1 root 0x61500000010e
0-18446744073709551615: node 0x615000000100 depth 0 type 1 parent 0x7ffd796e3e71 contents: (nil) 290 0xd 296 (nil) 18446744073709551615 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 (nil) 0 0x2
  0-290: (nil)
  291-296: value 6 (0x6) [0xd]
  297-18446744073709551615: (nil)
```

到这里，我想我也基本理解了xarray和maple tree之间的区别了。
