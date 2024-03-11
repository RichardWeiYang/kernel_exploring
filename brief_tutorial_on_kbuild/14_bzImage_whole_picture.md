在[启动镜像bzImage的前世今生][1]我们看到bzImage由setup.bin和vmlinux.bin两部分组成。

经过对这两部分的探索后，将整体展开在一起。希望有一个比较清晰的全景。


```
                                     *
                                     | <-- vmlinux.lds.S
                                     |
                                   vmlinux
                                     |
                                     | <-- objdump
                                     |
                             arch/x86/boot/compressed/vmlinux.bin
                                     |
                                     | <-- compress
                                     |
                             arch/x86/boot/compressed/vmlinux.bin.zst
                                     |
                                     | <-- mkpiggy
                                     |
                             arch/x86/boot/compressed/piggy.S
                                     |
                                     |
                                     |  arch/x86/boot/compressed/*
       arch/x86/boot/*                \  /
              |                        \/
              | <-- setup.ld            | <-- vmlinux.lds
              |                         |
              |                         v
              |              arch/x86/boot/compressed/vmlinux
              |                         |
              |                         | <-- objcopy
              |                         |
              v                         v
    arch/x86/boot/setup.bin  arch/x86/boot/vmlinux.bin  
                   \         /
                    \       /
               arch/x86/boot/bzImage
```

其中几个链接脚本具体是：

* vmlinux.lds.S -> arch/x86/kernel/vmlinux.lds.S
* setup.ld      -> arch/x86/boot/setup.ld
* vmlinux.lds   -> arch/x86/boot/compressed/vmlinux.lds

[1]: /brief_tutorial_on_kbuild/07_rules_for_bzImage.md
