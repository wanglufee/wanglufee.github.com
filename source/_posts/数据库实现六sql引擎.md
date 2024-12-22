---
title: 数据库实现六sql引擎
date: 2024-9-22 11:39:13
tags: 数据库
categories: rust，数据库
---
## sql引擎
sql引擎并不是一个常规意义上的组件，这里它主要用来作为执行器和底层存储引擎的中间部分，提供一些抽象，
可以方便我们拓展。

## 定义
```rust
pub trait Engine : Clone {
    type Transaction: Transaction;

    // 开启事务
    fn begin(&self) -> Result<Self::Transaction>;

    // 管理会话 
    fn session(&self) -> Result<Session<Self>> {
        Ok(Session{
            engine: self.clone(),
        })
    }
}
```
可以看到 sql引擎中主要有两个方法，一个用来开启事务，我们每条语句在执行时都会先开启一个事务，然后在
事务中进行具体的操作。另一个用来管理和客户端的会话。

对于事务的定义：
```rust
pub trait Transaction {
    // 提交事物
    fn commit(&self) -> Result<()>;

    // 回滚事物
    fn rollback(&self) -> Result<()>;

    // 创建行
    fn create_row(&mut self, table_name: String, row: Row) -> Result<()>;

    // 扫描表
    fn scan_table(&self, table_name: String) -> Result<Vec<Row>>;

    // DDL相关操作
    fn create_table(&mut self, table: Table) -> Result<()>;

    // 获取表信息
    fn get_table(&self, table_name: String) -> Result<Option<Table>>;
}
```
可以看到在事务的trait中我们定义了相应的操作，我们只需要用过开启一个事务，拿到事务的实例，就可以通过
事务内部的操作，来对数据库进行相应的操作。

对于 session 的定义：
```rust
pub struct Session<E: Engine> {
    engine: E,
}

impl<E: Engine> Session<E> {
    
    // 执行客户端 sql 语句
    pub fn execute(&mut self, sql: &str) -> Result<ResultSet> {
        match Parser::new(sql).parse()? {
            stmt => {
                // 开启一个事务
                let mut txn = self.engine.begin()?;
                
                match Plan::build(stmt).execute(&mut txn) {
                    Ok(result) => {
                        // 执行成功，提交事务
                        txn.commit()?;
                        Ok(result)
                    },
                    Err(err) => {
                        // 执行失败，回滚事务
                        txn.rollback()?;
                        Err(err)
                    }
                }
            }
        }
    }
}
```
在 session 中我们通过拿到客户端的 sql语句，对语句进行解析，然后生成执行节点，给执行器去执行，
在前面我们通过 Plan 的 build 方法来生成了一个执行节点，之后这个执行节点要传入执行器中，然后生成
每个 sql语句对应的执行器，才能真正执行，我们如何把执行节点给执行器呢？
## 执行节点到执行器到引擎
在这里我们首先为执行器定义一个根据执行节点生成对应执行器的 build 方法：
```rust
impl<T: Transaction> dyn Executor<T> {
    // 根据执行计划节点生成对应执行器
    pub fn build(node: Node) -> Box<dyn Executor<T>> {
        match node {
            Node::CreateTable { schema } => CreateTable::new(schema),
            Node::Insert { table_name, columns, values } => Insert::new(table_name, columns, values),
            Node::Scan { table_name } => Scan::new(table_name),
        }
    }
}
```
然后我们在执行节点上通过调用执行器的 build 方法来将执行节点转换为对应的执行器，并调用执行器上的 execute
 方法。
```rust
impl Plan {
    ...

    pub fn execute<T: Transaction>(self, txn: &mut T) -> Result<ResultSet> {
        <dyn Executor<T>>::build(self.0).execute(txn)
    }
}
```
这样我们就顺利将一个执行节点转换为一个执行器，并调用执行器上的 execute 方法。之后就是执行器通过
传入的事务，来进行实际的操作了。