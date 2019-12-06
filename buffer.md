## 硬盘IO缓存源码分析

### bread()源码分析
`bread()` 函数用于从磁盘中读取指定数据块的数据，参数 `dev` 是要读取的设备号，`block` 是要读取的数据块号，`size` 是要读取的数据长度。
```c
struct buffer_head * bread(kdev_t dev, int block, int size)
{
    struct buffer_head * bh;

    bh = getblk(dev, block, size); // 从缓存中查找缓存是否已经存在，如果不存在就创建一个新的
    if (buffer_uptodate(bh))       // 如果缓存是最新的，那么就直接返回
        return bh;
    ll_rw_block(READ, 1, &bh);     // 从磁盘中读取数据块的数据
    wait_on_buffer(bh);            // 等待数据读取完成
    if (buffer_uptodate(bh))       // 读取到的数据是否最新的? 是的话直接返回
        return bh;
    brelse(bh);
    return NULL;
}
```
