内核启动的过程不是一蹴而就的，而且其中还包含了不少让人不太好找的小秘密。

每次我看到相关的代码，都要从头再找一遍，感觉非常费事。这次干脆就记录下来，以备后用。

# INIT_CALL

在内核代码中经常会看到core_initcall(), subsys_initcall()这样xxx_initcall()的函数。

这些函数可以理解为c++中的构造函数，只是内核对这些函数做了分类，并且在特定的地方调用他们。

[INIT_CALLS的秘密][1]

# 内核参数

启动时我们可以通过[内核参数][2]来调整系统行为。

[1]: /bootup/01_init_call.md
[2]: /bootup/02_command_line_parse.md
