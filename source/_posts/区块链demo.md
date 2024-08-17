---
title: 区块链demo
date: 2024-02-10 12:56:55
tags: blockchain
categories: rust
---
# 基本介绍
## 基本概念
区块链是一个公开的，不可篡改的分布式数据库，每个人都有一个完整的或部分的副本，区块链应用部署于每个节点上，必须通过大多数节点的认同，才可以向区块链中添加内容。
## 区块
区块链由一个一个区块构成，所有信息也全部存在于区块之中，区块还包含了额外的其他信息，包括版本信息，时间戳，前一个区块的哈希值，以及当前区块的哈希值。

而在这个demo中我们的区块里只需要包含一些简单信息，我们目前的区块仅仅包含了如下信息：
```rust
pub struct Block {
    timestamp: u128,
    data: String,
    prev_block_hash: String,
    hash: String,
}
```
而在bitcoin中区块存在区块头，区块头是一个独立的结构体，时间戳，hash这些在区块头中，而区块头和交易信息等构成了区块。为了简便起见，我将信息和时间戳，hash等全部放入区块结构体中。

其中时间戳是记录当前区块的时间，内容则是每个节点需要向区块中记录的信息，前一个hash值，记录上一个区块的hash，当前hash则计算前三个字段的hash。

hash计算在bitcoin中是非常重要的一部分，正是因为hash计算才能保证bitcoin的安全，hash具有不可逆性，抗碰撞性，很难找到两个不同的数据具有同样的hash值，也无法通过hash值来反推数据，所以如果想要找到符合条件的hash值，那么唯一的方法就是不断计算，直到尝试出正确的解，正是由于这种特性，使得hash可以用来作为工作量的评估标准。在工作量证明部分我们会再看。

计算hash设置区块内容：
```rust
    fn set_hash(&mut self) -> Result<()> {
        // 计算时间戳
        self.timestamp = SystemTime::now()
                            .duration_since(SystemTime::UNIX_EPOCH)?
                            .as_millis();
        // 计算hash值
        let content = (self.timestamp,self.data.clone(),self.prev_block_hash.clone());
        let bytes = serialize(&content)?;
        let mut hasher = Sha256::new();
        hasher.input(&bytes);
        self.hash = hasher.result_str();
        Ok(())
    }
```
生成一个新区块：
```rust
    pub fn new_block(data: String,prev_block_hash: String) -> Result<Block> {
        let mut block = Block{
            timestamp: 0,
            data,
            prev_block_hash,
            hash: String::new(),
        };
        let _ = block.set_hash();
        Ok(block)
    }
```
## 区块链
有了区块，就可以开始构建区块链了，区块链中后一个区块指向前一个区块，根据时间顺序不停向后拓展，我们可以方便的获取最新的区块。这里我们用一个集合来保存区块。

```rust
pub struct Blockchain {
    blocks: Vec<Block>
}
```

在区块链中，第一个出现的块被称为创世区块，也是整个链的起点，初始化一个区块链，在初始化的过程中，需要生成一个创世区块。
为区块添加生成创世区块方法：
```rust
    pub fn new_genesis_block() -> Block {
        Block::new_block(String::from("创世区块"), String::new()).unwrap()
    }
```
初始化一个链：
```rust
    pub fn new() -> Blockchain {
        Blockchain{
            blocks: vec![Block::new_genesis_block()]
        }
    }
```
向区块链上添加区块:
```rust
    pub fn add_block(&mut self,data:String) -> Result<()> {
        let prev = self.blocks.last().unwrap();
        let new_block = Block::new_block(data, prev.get_hash())?;
        self.blocks.push(new_block);
        Ok(())
    }
```
ok基本的区块和区块链已经完成，来测试一下看结果如何。
```rust
fn main() {
    let mut bc = Blockchain::new();
    let _ = bc.add_block(String::from("手动添加一个区块"));
    let _ = bc.add_block(String::from("工作996真的太烦了"));
    print!("{:#?}",bc);
}
```
输出如下：
```bash
Blockchain {
    blocks: [
        Block {
            timestamp: 1710040858947,
            data: "创世区块",
            prev_block_hash: "",
            hash: "4641856e7ada2e715ac99e8caa4410bcf98a6a36c38cb1729217ae617facc07d",
        },
        Block {
            timestamp: 1710040858947,
            data: "手动添加一个区块",
            prev_block_hash: "4641856e7ada2e715ac99e8caa4410bcf98a6a36c38cb1729217ae617facc07d",
            hash: "85f61860660b0c1b73c24a433d821db1951c3f554d535a5f60aaf2bb4d98b0f7",
        },
        Block {
            timestamp: 1710040858947,
            data: "工作996真的太烦了",
            prev_block_hash: "85f61860660b0c1b73c24a433d821db1951c3f554d535a5f60aaf2bb4d98b0f7",
            hash: "fc90e189dc162fd68edfce5376c32fd7347e19a36deee2d8da2c588356081516",
        },
    ],
}
```
区块链雏形完成。

# 工作量证明
工作量证明在区块链中是非常重要的一部分，同时也保证了区块链的安全性。就如同我们在生活中一样，需要通过努力工作，才能拿到相应的报酬，而在区块链中每个矿工通过进行一定量的工作，来挖出一个区块，并从区块中得到一定的奖励。这样的机制就叫做工作量证明。在bitcoin中区块的生成速度被控制在10分钟左右，而工作则是寻找符合条件的hash值，只有找到符合条件的hash值，那么这个区块才算是一个合法的区块，所以对于矿工而言，就需要不断计算hash值。
## hash计算
通过hash函数可以计算给定数据的hash值，hash函数会输出一个固定长度的哈希值，hash具有几个特点：
- 从数据可以计算出hash，而从hash无法反推出数据。
- 每个数据的hash值唯一
- 对数据做很小的修改，hash值会变的完全不同
- 无法根据特定的hash值，去寻找生成这个hash值的数据

在区块链中hash被用来保证块的一致性，区块中包含有前一个区块的hash值，如果要修改一个区块，那么这个区块之后的所有区块都会被影响，因此使得篡改区块链上的数据所要付出的代价会变得非常大。

由于无法知道生成一个特定的hash值所需要的数据，所以如果要生成一个符合特定条件的hash值唯一的方法是计算，在bitcoin中有一个nonce值，这个值在生成区块的过程中可以被改变，由此来计算不同的hash，而根据对hash值的要求不同，计算难度也完全不同。

因此我们在区块的实现中添加一个nonce值，用来计算hash，添加一个TARGET，用来控制区块生成的难度。
```rust
const TARGET_HEXS: usize = 4;

/// 区块结构体
#[derive(Debug)]
pub struct Block {
    timestamp: u128,
    data: String,
    prev_block_hash: String,
    hash: String,
    nonce: i32,
}
```
其中TARGET是指生成的hash值的前TARGET位需要为0。

工作量证明即不停修改nonce值，计算出一个符合TARGET难度的hash值。
```rust
    fn run_proof_of_work(&mut self) -> Result<()> {
        while !self.validate()? {
            self.nonce += 1;
        }
        let data = self.prepare_hash_data()?;
        let mut hasher = Sha256::new();
        hasher.input(&data);
        self.hash = hasher.result_str();
        Ok(())
    }

    fn validate(&self) -> Result<bool> {
        let data = self.prepare_hash_data()?;
        let mut hasher = Sha256::new();
        hasher.input(&data);
        let hash = hasher.result_str();
        let mut vec1: Vec<u8> = Vec::new();
        vec1.resize(TARGET_HEXS, '0' as u8);
        Ok(hash[0..TARGET_HEXS] == String::from_utf8(vec1)?)
    }
```
看一下添加工作量证明之后的区块
```bash
Blockchain {
    blocks: [
        Block {
            timestamp: 1711241352238,
            data: "创世区块",
            prev_block_hash: "",
            hash: "0000f1ee1053aeff7a7184ae34907e79a4ce439007b26cff9f22e7a41c4ac4b8",
            nonce: 34797,
        },
        Block {
            timestamp: 1711241352358,
            data: "手动添加一个区块",
            prev_block_hash: "0000f1ee1053aeff7a7184ae34907e79a4ce439007b26cff9f22e7a41c4ac4b8",        
            hash: "0000946eae88f47da86be7d067853e5d4cdc0bef6223e689112c6217f4f8cd29",
            nonce: 118886,
        },
        Block {
            timestamp: 1711241353153,
            data: "工作996真的太烦了",
            prev_block_hash: "0000946eae88f47da86be7d067853e5d4cdc0bef6223e689112c6217f4f8cd29",
            hash: "00002a7035984cf78cde09c3d67669c6e4edb055a17d3a195470910c2bb3fd71",
            nonce: 34257,
        },
    ],
}
```
可以看到hash值的前TARGET位为0。