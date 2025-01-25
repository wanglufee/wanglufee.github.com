---
title: 数据库实现十一Bitcask初步定义
date: 2024-12-07 14:46:58
tags: 数据库
categories: rust，数据库
---
## 磁盘模型定义
首先来回顾一下 Bitcask ，它会将数据按照日志形式，逐条追加到文件末尾，并且生成一个键值关系存储在内存中，这个键值关系保存了键和对应数据在磁盘中位置的关系，当需要索引是，通过内存索引出对应的数据在磁盘上的位置，直接进行读取。

那么我们在磁盘上存储的数据条目需要有一个结构，这里一个条目中我们保存了 **键的长度，值的长度，键的内容，值的内容**，非常简单，并且前两个我们使用定长的数据字段，每个占据4个字节。总体如下：
```bash
|----------|----------|---------------|----------------|
| 键长     |  值长     |  键           |       值       |
|    4字节 |  4字节    |
```
这就是我们在磁盘中保存的数据条目，而内存中的索引关系我们使用 BTreeMap 来存储。
> set操作

set 操作我们需要在磁盘文件中追加一条条目，并且在内存中生成对应的索引。
> get操作
    
get操作我们先根据键拿到数据在磁盘中的位置，然后直接读取数据即可。
> delete操作

delete操作我们先将value长度设置为-1，表示是一个删除的值，之后在重启时，将其删除即可。

## 实现
之前是有定义一个存储引擎的 trait 的，我们的磁盘的存储引擎也需要实现那个 trait即可。
```rust
type KeyDir = BTreeMap<Vec<u8>, (u64,u32)>;
// 磁盘存储引擎
pub struct DiskEngine{
    keydir: KeyDir,
    log: Log,
}

pub struct Log {
    file: std::fs::File
}
```
这里使用 Log 结构体来对磁盘进行操作，而 KeyDir 类型则是我们在内存中的索引。

为存储引擎的结构体实现 trait：
```rust
impl super::engine::Engine for DiskEngine {
    type EngineIterator<'a> = DiskEngineIterator;

    fn set(&mut self, key: Vec<u8>, value: Vec<u8>) -> Result<()> {
        // 先写日志
        let (offset,size) = self.log.write_entry(&key, Some(&value))?;
        // 更新内存索引
        // 100----------------|-----150
        //                   130
        // val size = 20
        let val_size = value.len() as u32;
        // 条目中存入 value 在文件中的偏移以及 value 的长度
        self.keydir.insert(key, (offset + size as u64 - val_size as u64, val_size));
        Ok(())
    }

    fn get(&mut self, key: Vec<u8>) -> Result<Option<Vec<u8>>> {
        match self.keydir.get(&key) {
            Some((offset,val_size)) => {
                let val = self.log.read_value(*offset, *val_size)?;
                Ok(Some(val))
            },
            None => Ok(None)
        }
    }

    fn delete(&mut self, key: Vec<u8>) -> Result<()> {
        // 删除则写入None 并且从 keydir 中删除key条目
        self.log.write_entry(&key, None)?;
        self.keydir.remove(&key);
        Ok(())
    }

    fn scan(&mut self, range: impl std::ops::RangeBounds<Vec<u8>>) -> Self::EngineIterator<'_> {
        todo!()
    }
}
```
这里的 set 方法我们先将数据写入文件，并获取到条目的偏移以及长度。之后将键以及偏移插入到 KeyDir 中。 get 方法则通过查询数据在文件中的偏移，去读取数据。 delete 将数据内容置为空，并且从 KeyDir中删除对应的键值。

接下来是操作文件的 Log 的实现
```rust
impl Log {
    
    fn write_entry(&mut self,key: &Vec<u8>, value: Option<&Vec<u8>>) -> Result<(u64,u32)> {
        // 定位到文件末尾
        let offset = self.file.seek(SeekFrom::End(0))?;
        // 计算长度
        let key_size = key.len() as u32;
        let val_size = value.map_or(0, |v| v.len() as u32);
        let total_size = key_size + val_size + LOG_HEAD_SIZE;
        // 拿到写入缓存
        let mut writer = BufWriter::with_capacity(total_size as usize, &self.file);
        writer.write_all(&key_size.to_be_bytes())?;
        writer.write_all(&value.map_or(-1, |v| v.len() as i32).to_be_bytes())?;
        writer.write_all(&key)?;
        if let Some(v) = value {
            writer.write_all(&v)?;
        }
        writer.flush();
        // 返回相对应文件的偏移，和写入的总长度。
        Ok((offset, total_size))
    }

    fn read_value(&mut self,offset: u64, val_size: u32) -> Result<Vec<u8>> {
        // 定位到 value 所在位置
        self.file.seek(SeekFrom::Start(offset))?;
        // 定义存储 value 的 buf
        let mut buf = vec![0;val_size as usize];
        self.file.read_exact(&mut buf)?;
        Ok(buf)
    }
}
```
Log 目前提供了两个操作，一个写，一个读，在写的过程中我们要判断数据是否为 None ，如果为 None 那么说明这是一个删除操作，我们需要将文件中对应的值长度写为 -1，以此来表名数据无效。

这就是存储引擎的基本的方法，之后我们会实现存储引擎的启动和清理过程。