---
layout:       post
title:        "引入PageCache"
author:       "zhenbaii"
header-style: text
catalog:      true
categories: rCore-series
tags:
    - Rust
    - OS
    - file system
---
> 引入页缓存

在引入页缓存之前, 我们对于OSINode实现VfsInode, 都是直接透传到entity的对应方法, 也就是直接执行`self.entity.xxx()`.如果是普通文件的读写, 也就是对应efsnode后端, 相当于在之前每次用户态的读写请求是经过如下链路:

```jsx
User space -> Syscall -> OpenFile(inode) -> block cache(if hit)
```

即直接透传到了块设备对接层去读写文件:

```jsx
pub fn read_at(&self, offset: usize, buf: &mut [u8]) -> usize {
    let _fs = self.fs.lock();
    self.read_disk_inode(|disk_inode| disk_inode.read_at(offset, buf, &self.block_device))
}
```

虽然我们已经引入了块缓存来缓解访问内存和块设备的速度差距问题, 但是原先的这种设计还是会引入诸多问题, 特别是如下问题:

- 块和页是两种不同层次的抽象, 在处理文件的时候, 操作系统通常是在页抽象层面去做的, 现在的做法等于把两个抽象层次杂糅在一起, 不太解耦
- 就算不考虑普通文件的情况, 页缓存也是很有必要的, 因为他是许多特性和机制的基础, 如mmap, sync, 读写缓存等

因此, 我们引入页缓存来解决上述问题.

页缓存通俗来讲, 就是用来管理一个OSInode背后所管理的物理页的模块. 通常来说有两种做法, 一种做法是用一个全局的页缓存模块来管理所有inode的那些物理页, 通常会组织成一个多级结构, 第一级去索引inode, 下一级就去索引inode所管理的物理页. 另一种做法则是让每个inode自己维护自己的页缓存. 我们这里采取第二种做法:

```rust
pub struct OSInode {
    pub inode_id: u32, 
    pub dev_id: u32,
    pub entity: Arc<dyn VfsNode>,
    pub size: AtomicUsize,
    /* page cache */
    /* key: offset of the file;  val: PageCacheEntry */
    pub page_cache: InodePageCache
}

pub struct InodePageCache {
    // 如果用RwLock<BTreeMap<usize, PageCacheEntry>>, 多个进程持有&BTreeMap时, 顶多得到&PageCacheEntry
    // &PageCacheEntry是无法修改的
    // 如果要修改, 就要RwLock::write, 会锁住整个BTreeMap, 颗粒度太大了
    // map里存Arc<Mutex<_>>, 这样可以先 clone 出单页锁, 再释放外层读写锁
    frames: RwLock<BTreeMap<usize, Arc<Mutex<PageCacheEntry>>>>  /* 读>>写的频率, 因此用RwLock保护 */
}

// 这个会被一个Inode持有, 发生读写时, 一般优先来这里
pub struct PageCacheEntry {
    pub frame: Arc<FrameTracker>,    /* 物理帧 */
    pub state: PageState,            /* 页状态 */
}

#[derive(PartialEq)]
pub enum PageState {
    Sync,
    Dirty
}
```

每个OSInode下都有一个page_cache字段, 代表这个OSInode所管理的文件所拥有的物理页缓存. 上层在做修改或者读取时, 都会优先到达缓存层, 从缓存层返回结果, 或者就地修改缓存层某些物理页的内容. 这种机制的好处就是, 多次修改不会每次都触发磁盘的sync, 而是先修改缓存层中的内容, 等到时机合适, 如文件被关闭时, 统一sync到磁盘, 减少磁盘IO次数.

InodePageCache目前十分简洁, 里面就是用读写锁保护的一棵BTreeMap树, 树的key是page number, 值是PageCacheEntry. 其语义用通俗的语言讲就是: 这个文件的offset偏移处这个地址所处的页对应的物理页是PageCacheEntry. 或者用下图表示:

![image.png](/img/in-post/2026-06-03-引入PageCache/image-1.png)

InodePageCache维护的就是上方的一个个黄色的小长方形, usize是文件页号, 值是PageCacheEntry, 可以理解为物理页号. 可以发现, 页缓存也是可以处理文件大小非页对齐的文件的, 只不过这一页中有部分是垃圾数据, 这部分垃圾数据要妥善处理好, 防止污染到文件系统.

实际使用中, 用户会对文件的某处偏移, 即图上的offset发起调用请求, 这个请求就会被路由到PageCache缓存层, 这里会计算出offset对应的文件页号, 看这一页是否已经被缓存了. 如果被缓存了, 就直接返回其物理页号(如上图蓝色小长方形), 后续用户态的读取和修改都是在这个物理页上发生的. 如果未命中, 缓存层就会把需求继续往下路由到文件系统层, 让他把这一页所表征的内容返回来. 这部分逻辑都在get_or_load中:

```rust
// 表示获取或者加载inode代表的文件的第page_number页, 如果不在内存就要加载, 否则直接返回就好
pub fn get_or_load(&self, 
            page_number: usize, 
            loader: impl FnOnce() -> Result<PageCacheEntry, ErrorNo>) -> Result<Arc<FrameTracker>, ErrorNo>
{
    {
        let read_guard = self.frames.read();
        if let Some(entry_mutex) = read_guard.get(&page_number) {
            let guard = entry_mutex.lock();
            return Ok(Arc::clone(&guard.frame));
        }
    }

    let mut write_guard = self.frames.write();
    if let Some(entry_mutex) = write_guard.get(&page_number) {
        let guard = entry_mutex.lock();
        return Ok(Arc::clone(&guard.frame));
    }

    let entry = loader()?;
    let frame = Arc::clone(&entry.frame);
    write_guard.insert(page_number, Arc::new(Mutex::new(entry)));
    Ok(frame)
}
```

上层对接缓存层也十分简单, 就是让OSInode的read和write都走缓存层提供的方法, 而不是直接透传到self.entity:

```rust
fn read_at(&self, offset: usize, buf: &mut [u8]) -> Result<usize, ErrorNo> {
    if buf.is_empty() {
        return Ok(0);
    }

    let file_size = self.size.load(Ordering::Relaxed);
    if offset >= file_size {
        return Ok(0);
    }

    let read_len = buf.len().min(file_size - offset);
    let mut copied = 0usize;
		// 用户的offset和buf长度可能是任意的, 因此要按照页为单位分割好每次动作, 交给缓存层
    while copied < read_len {
        let current_offset = offset + copied;
        let page_number = current_offset / PAGE_SIZE;
        let page_offset = current_offset % PAGE_SIZE;
        let chunk_len = (PAGE_SIZE - page_offset).min(read_len - copied);
        let page_start = page_number * PAGE_SIZE;

        let frame = match self.page_cache.get_or_load(page_number, || {
            let frame = frame_alloc().expect("failed to allocate inode page cache frame");
            let page_buf = frame.ppn.get_bytes_array();
            let bytes_read = self.entity.read_at(page_start, page_buf)?;
            if bytes_read == 0 {
                return Err(ErrorNo::ENOENT);
            }
            Ok(PageCacheEntry {
                frame,
                state: PageState::Sync,
            })
        }) {
            Ok(f) => f,
            Err(_) => break,
        };

        let page_buf = frame.ppn.get_bytes_array();
        buf[copied..copied + chunk_len]
            .copy_from_slice(&page_buf[page_offset..page_offset + chunk_len]);
        copied += chunk_len;
    }
    Ok(copied)
}

fn write_at(&self, offset: usize, buf: &[u8]) -> Result<usize, ErrorNo> {
    if buf.is_empty() {
        return Ok(0);
    }

    let file_size = self.entity.file_size()?;
    let write_len = buf.len();
    let mut written: usize = 0;
    while written < write_len {
        let current_offset = offset + written;
        let page_idx = current_offset / PAGE_SIZE;
        let page_offset = current_offset % PAGE_SIZE;
        let chunk_len = (PAGE_SIZE - page_offset).min(write_len - written);
        let page_start = page_idx * PAGE_SIZE;

        let frame = self.page_cache.get_or_load(page_idx, || {
            let frame = frame_alloc().expect("failed to allocate inode page cache frame");
            let page_buf = frame.ppn.get_bytes_array();
            let bytes_read = self.entity.read_at(page_start, page_buf).unwrap_or(0);
            if page_start >= file_size || bytes_read == 0 {
                page_buf.fill(0);
            } else if bytes_read < PAGE_SIZE {
                page_buf[bytes_read..].fill(0);
            }
            Ok(PageCacheEntry {
                frame,
                state: PageState::Sync,
            })
        })?;

        let page_buf = frame.ppn.get_bytes_array();
        page_buf[page_offset..page_offset + chunk_len]
            .copy_from_slice(&buf[written..written + chunk_len]);
        written += chunk_len;
        self.page_cache.mark_dirty(page_idx);
    }
    let new_size = offset + write_len;
    if new_size > self.size.load(Ordering::Relaxed) {
        self.size.store(new_size, Ordering::Relaxed);
    }
    Ok(write_len)
}
```

我们可以用一张图来表示上面两个函数的实现逻辑:

![image.png](/img/in-post/2026-06-03-引入PageCache/image-2.png)

在关闭文件时, 我们通过在OpenFile上实现的Drop trait来自动实现物理页的回收:

```rust
// 关闭文件的时候, 自动执行同步
impl Drop for OpenFile {
    fn drop(&mut self) {
        let inner = self.inner.exclusive_access();
        inner.inode.sync_cache();
    }
}

pub fn sync_cache(&self) {
    let file_size = self.size.load(Ordering::Relaxed);
    self.page_cache.sync_all(|page_number, data| {
        let page_start = page_number * PAGE_SIZE;
        if page_start >= file_size {
            return;
        }
        let valid_len = PAGE_SIZE.min(file_size - page_start);  // 处理EOF没有页对齐的情况
        let _ = self.entity.write_at(page_start, &data[..valid_len]);
    });
}
```

实现到这里, 我们已经成功地引入了页缓存层, 后续mmap的实现以及其他机制的实现都会大量依赖这个模块.

在其他文章中, 我们会讲一下加在这个模块上的各种锁的设计.