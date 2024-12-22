---
title: 数据库实现九sql引擎层KV引擎
date: 2024-11-02 16:07:26
tags: 数据库
categories: rust，数据库
---
## sql层KV引擎
上一节我们实现了底层的基于内存的存储引擎，这一节我们来实现一个sql层的 KV 的引擎，它会以键值对的形式存储数据，并且调用底层内存的存储引擎。
### 定义
```rust
pub struct KVEngine<E : StorageEngein>{
    pub kv : storage::mvcc::Mvcc<E>,
}
```
这里的 KV引擎是对 mvcc 的封装，我们的 sql引擎层需要做的事情主要是逻辑上的判断，比如，表是否已经存在，类型是否匹配等，校验完成之后，对数据进行序列化，然后直接调用 mvcc 提供的接口，将数据传递给底层的存储引擎即可。

对于 sql引擎 trait 的实现
```rust
impl<E: StorageEngein> KVEngine<E>  {
    pub fn new(engine: E) -> Self{
        Self{
            kv: storage::mvcc::Mvcc::new(engine)
        }
    }
}

impl<E : StorageEngein> Clone for KVEngine<E> {
    fn clone(&self) -> Self {
        Self { kv: self.kv.clone() }
    }
}

impl<E : StorageEngein> Engine for KVEngine<E> {
    type Transaction = KVTransaction<E>;

    fn begin(&self) -> Result<Self::Transaction> {
        Ok(Self::Transaction::new(self.kv.begin()?))
    }
}
```
这里主要实现了开启事务的 begin 方法，返回一个具体的 KVTransaction，这个事务中也是逻辑实现的地方。
### 事务定义和实现
```rust
pub struct KVTransaction<E : StorageEngein> {
    txn: storage::mvcc::MvccTransaction<E>,
}
```
这里的事务则是对 MvccTransaction 的封装。MvccTransaction 提供了接口去调用存储引擎的功能。

具体校验逻辑的实现：
```rust
impl<E : StorageEngein> Transaction for KVTransaction<E> {
    fn commit(&self) -> Result<()> {
        Ok(())
    }

    fn rollback(&self) -> Result<()> {
        Ok(())
    }

    fn create_row(&mut self, table_name: String, row: Row) -> Result<()> {
        let table = self.must_get_table(table_name.clone())?;
        // 检查类型有效性
        for (i,col) in table.columns.iter().enumerate() {
            match row[i].datatype() {
                None if col.nullable => {},
                None => return Err(Error::Internel(format!("column {} cannot be null",col.name))),
                // 判断数据的类型和列的类型是否匹配
                Some(dt) => {
                    if dt != col.datatype {
                        return Err(Error::Internel(format!("column {} type mismatched",col.name)));
                    }
                },
            }
        }

        // 存放数据
        // 暂时以第一列作为主键
        let id = Key::Row(table_name, row[0].clone());
        let value = bincode::serialize(&row)?;
        self.txn.set(bincode::serialize(&id)?, value)?;

        Ok(())
    }

    fn scan_table(&self, table_name: String) -> Result<Vec<Row>> {
        let perfix = KeyPerfix::Row(table_name.clone());
        let results = self.txn.scan_prefix(bincode::serialize(&perfix)?)?;

        let mut rows = Vec::new();
        for result in results {
            let row: Row = bincode::deserialize(&result.value)?;
            rows.push(row);
        }
        Ok(rows)
    }

    // 创建表，此处去调用底层存储引擎的接口
    fn create_table(&mut self, table: Table) -> Result<()> {
        // 判断表是否已经存在
        if self.get_table(table.name.clone())?.is_some() {
            return Err(Error::Internel(format!("table {} already exists",table.name)));
        }
        // 判断表的有效性
        if table.columns.is_empty() {
            return Err(Error::Internel(format!("table {} has no columns",table.name)));
        }
        // 将表名序列化作为键，将整张表序列化作为值
        let key = Key::Table(table.name.clone());
        let value = bincode::serialize(&table)?;
        self.txn.set(bincode::serialize(&key)?, value)?;
        Ok(())
    }

    fn get_table(&self, table_name: String) -> Result<Option<Table>> {
        let key = Key::Table(table_name);
        Ok(self.txn.get(bincode::serialize(&key)?)?
                .map(|v| bincode::deserialize(&v))
                .transpose()?)
    }
}
```
这里再创建表时需要对创建表的输入进行判断，表是否已经存在，表是否有符合要求的列，其他几个接口也同样做了对应的逻辑判断，以保证正确性。

## 小结
至此，我们的整个流程就完全打通了。

1. 通过解析用户传入的sql语句，首先进行词法分析，得到对应的 Token
2. 然后用 Token 进行语法分析，生成对应的语法树
3. 开启一个事务
4. 通过语法树生成对应的执行节点
5. 执行节点通过执行器提供的 build 方法，构建出相应的执行器
6. 执行器拿到事务，并进行执行
7. 执行器在执行的过程中调用 sql引擎层的事务的接口
8. 在 sql引擎层进行相应的逻辑判断，无误后调用 mvcc事务的接口
9. mvcc事务最后调用存储引擎接口将数据存储