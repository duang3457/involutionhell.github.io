# 理论部分
### 什么是数据库连接池？
维持管理一定数量连接的池式结构

### 数据库连接池解决了什么问题？
复用资源，提升并发处理sql能力

### 同步连接和异步连接的区别
#### 同步连接：通过接口的返回值接收数据库返回值
1. auto& res = db -> Query(SQLStr);
2. 同步连接池：阻塞发起请求的线程
3. 使用场景，初始化服务器资源（资源初始化先后问题）

![](https://cdn.nlark.com/yuque/0/2025/png/43055607/1758776865390-8b964d30-9def-423c-af9a-54f583da3d4d.png)

#### 异步连接：通过回调函数接收数据库返回值
1. db -> AsyncQuery(SQLStr, **callback**);
2. 异步连接池：不阻塞发起请求的线程，阻塞线程池中执行任务线程

![](https://cdn.nlark.com/yuque/0/2025/png/43055607/1758780277485-80c0a119-2f90-4831-919a-92e6b811ba5e.png)



### 单条mysql连接过程
![](https://cdn.nlark.com/yuque/0/2025/png/43055607/1758780851941-b8fa1e52-7083-4b4f-8aae-6a3afcb0ccdf.png)

### 现有两个mysql驱动
    1. libmysqlclient 纯c实现
    2. libmysqlcppconn c++实现，使用了异常机制

上面是阻塞io

### 该连接池中所用到libmysqlcppconn中的类
```cpp
namespace sql 
{
    class Driver;
    class Connection;
    class SQLException;
    class ResultSet;
}
```

#### Driver类
Driver 由库中全局工厂函数（自由函数）get_driver_instance()返回，表示全局唯一的驱动实例。（并不是创建，只是get。因为我们要在池子里开多个conn，每个线程下的conn->open都要get一次driver实例）。

那么dirver实例是在什么时候创建的呢？懒加载：第一个调用时被创建，后面只是调用。

```cpp
sql::Driver *driver_ = get_driver_instance();
```

#### Connection类
使用 dirver_ -> connect() 获取一条连接。

```cpp
sql::Connection *conn_ = driver_->connect(info_.url, info_.user, info_.password);
```

在我们下面的项目代码中，线程池初始化阶段InitPool中，会

conn_有一些方法

```cpp
conn_->setSchema(info_.database);   // 这是我们下面代码用到的方法，等同于指令 use XXXdb;

语句相关
createStatement(): 创建简单语句对象，用于 executeQuery/executeUpdate
prepareStatement(sql): 创建预处理语句，支持 ? 占位符，防注入、可复用
事务与提交
setAutoCommit(bool on): 设置自动提交（默认 true）
getAutoCommit(): 查询自动提交状态
commit(): 提交当前事务（在 setAutoCommit(false) 时生效）
rollback(): 回滚当前事务
数据库与元信息
setSchema(db): 切换默认数据库（等价于 USE db）
getSchema(): 获取当前默认数据库
getMetaData(): 获取元数据接口（库、表、列等信息）
连接状态与关闭
isValid(): 判断连接是否有效（部分版本支持 isValid(timeout)）
isClosed(): 是否已关闭
close(): 关闭连接（你在 Close() 里有调用）
事务隔离级别
setTransactionIsolation(enum): 设置隔离级别（如 READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE）
getTransactionIsolation(): 获取当前隔离级别
客户端/连接选项
setClientOption(key, value): 设置客户端选项（不同版本键名支持不同，比如字符集、超时等）
getClientOption(key, outValue): 读取选项
```

#### ResultSet类
 内部绑定：  

+ 数据缓冲区；
+ 连接对象；
+ `sql::ResultSetMetaData`（列结构）；
+ 当前游标位置；
+ 状态信息（是否结束、是否关闭）。

#### SQLException类
```cpp
try {
        sql::Statement *stmt = conn_->createStatement();
        return stmt->executeQuery(sql);
    } catch (sql::SQLException &e) {   // 表示错误类型
        HandlerException(e);
}
```



### mysql驱动需要实现的内容(同官方驱动)
    3. connect
    4. read/recv
    5. write/send
    6. read/recv

上面是阻塞io

# 代码部分
## 设计图
```cpp
┌─────────────────────────────────────────────────────────────────┐
│                          应用层                                  │
│  main() → MySQLConnPool::Query() → AsyncProcessor               │
├─────────────────────────────────────────────────────────────────┤
│                       连接池管理层                               │
│         MySQLConnPool (单例模式 + 多数据库支持)                   │
├─────────────────────────────────────────────────────────────────┤
│                      任务调度层                                  │
│    BlockingQueue<SQLOperation*> (线程安全队列)                   │
├─────────────────────────────────────────────────────────────────┤
│                      工作线程层                                  │
│  MySQLWorker threads (每个连接一个工作线程)                       │
├─────────────────────────────────────────────────────────────────┤
│                      连接管理层                                  │
│     MySQLConn (封装MySQL C++ Connector)                         │
├─────────────────────────────────────────────────────────────────┤
│                      异步回调层                                  │
│  QueryCallback + std::future/promise (异步结果处理)              │
└─────────────────────────────────────────────────────────────────┘
```

## 流程与对应类
![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/43055607/1760850108252-24bbc55b-509c-4e9e-a41c-709aa55eca9f.jpeg)

由图可见，MySQLConnPool是一个单例，每个库（database）对应一个连接池。



## 类的调用图
![画板](https://cdn.nlark.com/yuque/0/2025/jpeg/43055607/1760850406668-74d47d75-7166-4278-a615-e9090ca0dcf8.jpeg)

### 连接池类 MySQLConnPool
#### 成员
##### BlockingQueue<SQLOperation *> *task_queue_;
连接池需要一个阻塞队列，在initPool时new一个

##### std::vector<MySQLConn *>  pool_;
维护一个连接集合

##### static std::unordered_map<std::string, MySQLConnPool *> instances_;
每个pool实例对应一个database。

注意这个要加static，因为这个对照表不是绑定在某一个pool实例上，而是放在全局空间上。（假设不加static，让其绑定在某个实例上，那我们每次创建/删除新db的pool时，都要全局更新所有pool，相当麻烦还会有问题）（面试中会问到static的作用域之类的问题，这里就是一个很好的例子）

##### std::string database_;
该pool对应的db名

#### 接口
##### *GetInstance 
单例模式需要的接口。因为我们要判断该db的pool是否已创建过，

```cpp
MySQLConnPool *MySQLConnPool::GetInstance(const std::string &db) {
    if (instances_.find(db) == instances_.end()) {
        instances_[db] = new MySQLConnPool(db);
    }
    return instances_[db];
}
```

##### InitPool 
创建阻塞队列，创建指定数量连接

```cpp
void MySQLConnPool::InitPool(const std::string &url, int pool_size) {
    task_queue_ = new BlockingQueue<SQLOperation *>();
    for (int i = 0; i < pool_size; ++i) {
        MySQLConn *conn = new MySQLConn(url, database_, *task_queue_);
        conn->Open();
        pool_.push_back(conn);
    }
}
```

##### Query(sql, functional)   
生成SQLOperation并放入阻塞队列，SQLOperation与普通string sql_的区别就在于多了一个promise。

promise有get_future方法，先返回一个占位future(此时还没有有效值，因为sql还没执行完（假设执行的速度很慢）)，后面可以轮询检查每一个future对应的回调结果（在main里做的），是否Ready(在QueryCallback中实现的future_.wait_for)。Ready了则用future_.get()取出结果。

```cpp
QueryCallback MySQLConnPool::Query(const std::string &sql, std::function<void(std::unique_ptr<sql::ResultSet>)> &&cb) {
    SQLOperation *op = new SQLOperation(sql);
    auto future = op->GetFuture();
    task_queue_->Push(op);
    return QueryCallback(std::move(future), std::move(cb));
}
```

##### ~MySQLConnPool  
// 析构也做一下处理

```cpp
MySQLConnPool::~MySQLConnPool() {
    if (task_queue_)
        task_queue_->Cancel();
    for (auto conn : pool_) {
        delete conn;
    }
    if (task_queue_) {
        delete task_queue_;
        task_queue_ = nullptr;
    }
    pool_.clear();
}
```

### SQL执行任务类 SQLOperation
#### 成员
+ 一条sql语句  
+ 一个promise<std::unique_ptr<sql::ResultSet>>   // 接受结果

> 为什么用promise？ `std::promise`是异步结果的“占位符”  。 数据库查询往往是**耗时操作**，我们通常不希望阻塞主线程（例如事件循环、工作线程池中的调度线程）。于是我们把任务交给后台线程执行  。
>

#### 接口
##### Execute
```cpp
// 执行接口，异步回调得到结果
// 因为SQL语句的执行可能是个很耗时的操作，所以要基于promise来使用
void SQLOperation::Execute(MySQLConn *conn)
{
    auto result = conn->Query(sql_); 
    // 在Query中我们写了：
    // return stmt->executeQuery(sql); 这里会将该线程阻塞在这
    promise_.set_value(std::unique_ptr<sql::ResultSet>(result)); // 上面返回得到结果后才能同步设置value，并改变标志为ready
}
```

##### GetFuture
```cpp
std::future<std::unique_ptr<sql::ResultSet>> GetFuture() {
    return promise_.get_future();
}
```



### MySQL连接类 MySQLConn
#### 成员
##### sql::Driver *driver_
来自官方库

##### sql::Connection *conn_;
来自官方库

##### MySQLWorker *worker_; 
一个worker线程

##### MySQLConnInfo info_; 
连接信息

##### void HandlerException(sql::SQLException &e);
// 处理错误

```cpp
void MySQLConn::HandlerException(sql::SQLException &e)
{
    if (e.getErrorCode() != 0)
    {
        std::cerr << "# ERR: SQLException in " << __FILE__;
        std::cerr << "(" << __FUNCTION__ << ") on line " << __LINE__ << std::endl;
        std::cerr << "# ERR: " << e.what();
        std::cerr << " (MySQL error code: " << e.getErrorCode();
        std::cerr << ", SQLState: " << e.getSQLState() << " )" << std::endl;
    }
}
```

#### 接口
##### Open  
// 将connect绑定上

```cpp
int MySQLConn::Open()
{
    int err = 0;
    try {
        driver_ = get_driver_instance();
        conn_ = driver_->connect(info_.url, info_.user, info_.password);
        if (!conn_) {
            return -1;
        }
        conn_->setSchema(info_.database);
    } catch (sql::SQLException &e) {
        HandlerException(e);
        err = e.getErrorCode();
    }
    return err;
}
```

##### Close  
// 将连接关闭

```cpp
void MySQLConn::Close()
{
    if (conn_) {
        conn_->close();
        delete conn_;
        conn_ = nullptr;
    }
}
```

##### Query  
// 执行Query 调用的是官方库中sql::Statement来执行

```cpp
sql::ResultSet* MySQLConn::Query(const std::string &sql)
{
    try {
        sql::Statement *stmt = conn_->createStatement();
        return stmt->executeQuery(sql);
    } catch (sql::SQLException &e) {
        HandlerException(e);
    }
    return nullptr;
}
```

### 工作线程类 MySQLWorker
#### 成员
##### MySQLConn *conn_
依赖一条连接去访问mysql

##### std::thread worker_
一个worker一个线程

##### BlockingQueue<SQLOperation *> &task_queue_
上面的线程从队列中取出任务

##### void Worker()
这个私有方法实现了从阻塞队列里搬运（Pop）出任务（SQLOperation）的职责。由Start接口调用。

#### 接口
##### start
开线程，调用Worker()，Pop出下一个Operation，并执行（op->Execute(conn_)）。

```cpp
void MySQLWorker::Start()
{
    worker_ = std::thread(&MySQLWorker::Worker, this);
}
```

##### stop
同步等待工作线程结束并回收资源

```cpp
void MySQLWorker::Stop()
{
    if (worker_.joinable()) {
        worker_.join();
    }
}
```



### 回调类 QueryCallback
#### 成员
##### std::future<std::unique_ptr<sql::ResultSet>> future_;
一个future对象，在QueryCallback中检查是否ready。Ready了就把结果get到，并move进cb_中，执行cb_函数。

##### std::function<void(std::unique_ptr<sql::ResultSet>)> cb_;
拿到ResultSet后，该调用的函数

#### 接口
##### 构造
```cpp
QueryCallback(std::future<std::unique_ptr<sql::ResultSet>> &&future, std::function<void(std::unique_ptr<sql::ResultSet>)> &&cb)
        : future_(std::move(future)), cb_(std::move(cb)){}
```

##### bool InvokeIfReady()
```cpp
bool InvokeIfReady() {
    if (future_.wait_for(std::chrono::seconds(0)) == std::future_status::ready) {
        cb_(std::move(future_.get()));
        return true;
    }
    return false;
}
```

