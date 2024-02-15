在进入start_kernel之前，那真的是一片黑暗的路程。因为好多都是用汇编写的，对我来说简直就是抓瞎。

在[bzImage的全貌][1]中我们看过了make install时安装的bzImage的组成部分。而start_kernel在这几个组成部分的最后一点。


```
                                     *
                                     | <-- vmlinux.lds.S
                                     |
                                   vmlinux
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

这次我们先来说说，被安装的内核是通过哪些步骤走到start_kernel的。

**setup.bin**:
```
_start -> main -> go_to_protected_mode -> protected_mode_jump(boot_params.hdr.code32_start, )
```

**vmlinux.bin**:
```
startup_32 -> startup_64 -> extract_kernel ->
```

**vmlinux**:
```
startup_64 -> initial_code -> x86_64_start_kernel -> x86_64_start_reservations
```

[1]: /brief_tutorial_on_kbuild/14_bzImage_whole_picture.md