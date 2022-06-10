虽然gmail够大，但架不住社区邮件太多。突然发现两年前自己写过相关的一组patch，但是没有被接受。
也不知道发生了什么。。。

自己的邮箱里已经删了，但是web页面查邮件还是不太友好，想要下载到本地用mutt来看。搜了一下果然可以。

# 找到Message-ID

通过google找到邮件，可以用title。找网址是https://lore.kernel.org的。
点进去，里面有个raw选项。再点进去

# 找到Message-ID的邮件

lkml提供了根据Message-ID[列出邮件的功能][1]。

用刚才找到的Message-ID拼接出url

http://lore.kernel.org/lkml/<Message-ID>/

# 下载整个会话

找到 Thread overview: 这一行，有一个mbox.gz按钮。
ok，这样就可以将整个会话下载到本地了～

[1]: https://lore.kernel.org/lkml/_/text/help/
