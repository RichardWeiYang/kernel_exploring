Folio 是2020年Matthew引入的，用来代表物理内存的新结构。


```
commit 7b230db3b8d373219f88a3d25c8fbbf12cc7f233
Author: Matthew Wilcox (Oracle) <willy@infradead.org>
Date:   Sun Dec 6 22:22:48 2020 -0500

    mm: Introduce struct folio
    
    A struct folio is a new abstraction to replace the venerable struct page.
    A function which takes a struct folio argument declares that it will
    operate on the entire (possibly compound) page, not just PAGE_SIZE bytes.
    In return, the caller guarantees that the pointer it is passing does
    not point to a tail page.  No change to generated code.
```

相比与page/compound_page，folio的作用是把逻辑上相关的几个page struct用一个folio结构来呈现。

而且Matthew的目标是替换调page struct。

其实folio的样子说起来很简单，就是几个page struct的组合，但是具体那个成员应该放哪里需要多年内核的功力。

我们来看一下结构体，有一个直观的印象：


```
struct folio {
	/* private: don't document the anon union */
	union {
		struct {
	/* public: */
			unsigned long flags;
			...
		};
		struct page page;
	};
	union {
		struct {
			unsigned long _flags_1;
			unsigned long _head_1;
			...
		};
		struct page __page_1;
	};
	union {
		struct {
			unsigned long _flags_2;
			unsigned long _head_2;
			...
		};
		struct page __page_2;
	};
	union {
		struct {
			unsigned long _flags_3;
			...
		};
		struct page __page_3;
	};
};
```

我们只看每个union中第二个成员，就可以看出folio可以理解为一个page[4]的数组。
