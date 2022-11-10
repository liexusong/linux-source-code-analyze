# eBPF 源码分析

eBPF 使用 BPF_PROG_RUN() 宏来运行 eBPF 程序，其定义如下：

```c
#define BPF_PROG_RUN(filter, ctx) (*filter->bpf_func)(ctx, filter->insnsi)
```

一般来说，filter参数的类型为：`struct bpf_prog`，其定义如下：

```c
struct bpf_prog {
    ...
    u16                     jited:1,    /* 是否已经编译程JIT? */
    ...
    // 1. 如果是虚拟机执行，那么指向 __bpf_prog_run() 函数
    // 2. 如果是 JIT 执行，那么指向 eBPF 程序经过 JIT 转换后的可执行二进制字节码
    unsigned int            (*bpf_func)(const struct sk_buff *skb,
                                        const struct bpf_insn *filter);
    /* eBPF字节码 */
    union {
        struct sock_filter  insns[0];
        struct bpf_insn     insnsi[0];
    };
};
```

我们来看看 Linux 内核各个模块是怎么来执行 eBPF 程序的吧。

首先，我们来看看 perf 模块是怎么执行 eBPF 的。我们可以看到 trace_call_bpf() 函数中有以下一段代码：

```c
unsigned int trace_call_bpf(struct bpf_prog *prog, void *ctx)
{
    unsigned int ret;
    ...

    rcu_read_lock();
    ret = BPF_PROG_RUN(prog, ctx);
    rcu_read_unlock();
    ...

    return ret;
}
```

而 `trace_call_bpf()` 函数又被 `kprobe_perf_func()` 函数调用，如下代码所示：

```c
static void
kprobe_perf_func(struct trace_kprobe *tk, struct pt_regs *regs)
{
    struct trace_event_call *call = &tk->tp.call;
    struct bpf_prog *prog = call->prog;
    ...

    if (prog && !trace_call_bpf(prog, regs))
        return;

    ...
}
```

而 `kprobe_perf_func()` 又被 `kprobe_dispatcher()` 函数调用，其代码如下：

```c
static int kprobe_dispatcher(struct kprobe *kp, struct pt_regs *regs)
{
    struct trace_kprobe *tk = container_of(kp, struct trace_kprobe, rp.kp);
    ...

#ifdef CONFIG_PERF_EVENTS
    if (tk->tp.flags & TP_FLAG_PROFILE)
        kprobe_perf_func(tk, regs);
#endif

    return 0;
}
```

所以 perf 模块执行 eBPF 程序的调用链如下：

```c
kprobe_dispatcher()
   -> kprobe_perf_func()
        -> trace_call_bpf()
             -> BPF_PROG_RUN()
```

那么 `kprobe_dispatcher()` 函数在什么时候被调用呢？我们查看源码可以发现，内核并没有直接调用 `kprobe_dispatcher()` 函数的地方。那么这个函数是怎么被调用的呢？

这个问题涉及到 trace 模块，当我们使用以下命令设置一个 kprobe 事件时，将会触发调用 `create_trace_kprobe()` 函数。

```bash
> echo "r:kretprobe_func sys_write $retval" /sys/kernel/debug/tracing/kprobe_events
```

我们来看看 `create_trace_kprobe()` 函数的实现：

```c
static int create_trace_kprobe(int argc, char **argv)
{
    ...
    struct trace_kprobe *tk;

    tk = alloc_trace_kprobe(group, event, addr, symbol, offset, argc,
                            is_return);
    ...

    ret = register_trace_kprobe(tk);
    if (ret)
        goto error;
    return 0;
}
```

`create_trace_kprobe()` 函数主要完成 2 件事情：

* 调用 `alloc_trace_kprobe()` 函数创建一个 `trace_kprobe` 结构，并且初始化它。
* 调用 `register_trace_kprobe()` 函数对事件进行注册。

我们先来看看 `alloc_trace_kprobe()` 函数做了什么事情：

```c
static struct trace_kprobe *
alloc_trace_kprobe(const char *group, const char *event, void *addr,
                   const char *symbol, unsigned long offs, int nargs,
                   bool is_return)
{
    struct trace_kprobe *tk;

    tk = kzalloc(SIZEOF_TRACE_KPROBE(nargs), GFP_KERNEL);
    ...

    if (is_return)
        tk->rp.handler = kretprobe_dispatcher;
    else
        tk->rp.kp.pre_handler = kprobe_dispatcher;

    ...
    return tk;
}
```

从 `alloc_trace_kprobe()` 函数的代码可以看出，其将 `trace_kprobe` 结构的 `rp.kp.pre_handler` 成员设置为 `kprobe_dispatcher()` 函数。我们可以看看 `trace_kprobe` 结构的定义：

```c
struct trace_kprobe {
    ...
    struct kretprobe rp;
    ...
};

struct kretprobe {
    struct kprobe kp;
    ...
};

struct kprobe {
    ...
    kprobe_pre_handler_t pre_handler;
    ...
};
```

看到这里，`kprobe_dispatcher()` 函数何时被调用已经有点眉目了。因为在 `alloc_trace_kprobe()` 函数中，我们看到了 `kprobe_dispatcher()` 函数的踪影。

我们接着来看 `register_trace_kprobe()` 函数的实现。

通过分析，可以看到 `register_trace_kprobe()` 最终会调用 `register_kprobe()` 函数注册一个 `kprobe` 跟踪点，跟踪点的回调就是 `register_kprobe()` 函数。调用链如下：

```c
register_trace_kprobe()
  -> __register_trace_kprobe()
       -> register_kprobe()
```

我们在 kprobe 源码分析一文中分析过，当跟踪点被触发时，首先会调用 `pre_handler` 成员指向的函数来处理。而通过上面的分析可知，`pre_handler` 成员指向了 `kprobe_dispatcher()` 函数。所以当跟踪点被触发时，将会调用 `kprobe_dispatcher()` 函数来进行处理。调用链如下：

```c
do_int3()
  -> kprobe_int3_handler()
       -> kprobe_dispatcher()
```




