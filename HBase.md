# CHAPTER I
## 列式存储数据库
1. 基于假设：对于特定查询，并不是所有值都是必须的。
2. 优点：列的数据类型天生相似，比按行存储的结构聚集在一起的数据更利于压缩。

## HBase与传统列式数据库的不同
    传统列式数据库比较适合实时存取数据的场景，HBase比较适合键值对的数据存取，或有序的数据存取。
## 论文
    BigTable: A Distrubuted Storage System for Structured Data
## HBase数据表的组织形式
1. 行序按照二进制逐字节从左到右以此对比每一个行键，如row12排在row2前面。
2. 行键总是唯一的。
3. 一行由若干列组成，若干列构成列族(column family)。
4. 列族有利于构建数据的语义边界或局部边界，还有助于设置特性（如压缩），或者指示它们存在内存中。
5. 一个列族的所有列存储在同一个底层的存储文件（HFile）里。
6. 存取模式(Table, RowKey, Family, Column, Timstamp)-> Value   
    SortedMap<RowKey, List<SortedMap<Column, List<Value, Timestamp>>>>
7. 行数据的存取操作是原子的，可以读写任意数目的列。
8. HBase默认保留3个版本的数据，按照版本的时间戳降序存储。
## 自动分区
1. HBase通过RegionServer对数据库进行自动分区，从而做到负载均衡。Region本质上是以行键排序的连续存储区间。如果Region太大，系统会将他们动态拆分，相反地，就把多个region合并，以减少存储文件数量。
2. 一张表初始只有一个region，用户插入数据时，系统检测region大小，当超过限制，从中间键（middle key，region中间那个键）处将该region切分为两个大致相等的子region。
3. 每个region由一个region服务器（region server）加载，每台region服务器可以同时加载多个region。每张表由多个region组成，这些region可以由不同region服务器加载。
4. region过大时触发自动分区（autosharding），当服务于某个region的服务器负载过大或发生错误进而导致不可用时，系统会将该region移至其他服务器，从而快速恢复region，并获得细粒度的负载均衡。
# CHAPTER III
## HTable
1. HBase的主要客户端接口由org.apche.hadoop.hbase.client包中的HTable类提供。
2. 所有修改数据的操作保证了行级别的原子性，当许多客户端需要修改同一行数据，会产生问题。因此应当尽量使用批处理更新来减少单独操作同一行数据的次数。
3. 创建HTable是有代价的，每个实例都需要扫描.META.表，推荐一次创建一个，然后再生存期内复用这个对象。当需要使用多个HTable，应考虑使用HTablePool。
## HBaseConfiguration
static Configuration create()   
当尝试调用一个静态的create()方法时，代码会尝试使用当前java classpath来载入配置文件hbase-default.xml和hbase-site.xml。   
同样的，也可以简单地忽略外部的配置文件，直接在代码中设置hbase.zookeeper.quorum属性来创建一个不需要额外配置的客户端。   
Configuration config = HBaseConfiguration.create();   
config.set("hbase.zookeeper.quorum","ip1,ip2,ip3");   
***同时应当共享配置实例。***
## put操作与缓冲区
1. 每一个put操作实际是一个RPC操作，受往返时间限制，一秒钟最多只能完成1000次RPC往返相应。故而应当减少独立RPC调用，将多个put请求放先在客户端的缓冲区，以此来一次执行多个put操作。
2. 缓冲区默认为禁用，通过将自动刷写（auto flush）设置为false来激活缓冲区：   
    table.setAutoFlush(false)
3. 用户无需强制刷写缓冲区，因为一旦超出缓冲指定大小限制（默认为2mb），客户端会隐式地调用刷写命令，可以通过以下方法来配置缓冲区大小。  
    long getWriteBufferSize()
    void setWriteBufferSize(long writeBufferSize) throws IOException
4. 当需要强制把数据写到服务端时，使用以下方法：   
    void flushCommits() throws IOException
5. 隐式刷写会在调用put()或setWriteBufferSize()方法时触发。此外，当调用HTable的close()方法时也会无条件触发刷写。

## put列表
1. List<Put> puts = new ArrayList<Put>();
   Put put1 = new Put(Bytes.toBytes("row1"));
   put1.add(Bytes.toBytes("colfam1"),Bytes.toBytes("qual1"),Byes.toBytes("val1"));
   
   puts.add(put1);
   
   Put put2 = ...   
   ...   
   puts.add(put2);

   table.put(puts);
2. 对列族的检查在服务端完成，当向HBase插入一个错误的列族，会返回如下的错误信息：   
    NoSuchColumnFamilyException: 1 time,   
    服务端会遍历所有操作并设法执行它们，失败的会返回，客户端会用RetriesExhaustedWithDetailsException报告远程错误。对于错误列族，服务端的重试次数会自动设为1.
3. 在服务端失败的Put实例会被保存在本地写缓冲区中，下一次缓冲区刷写时会重试。可以通过HTable的getWriteBuffer()对它们进行访问和清除。
4. 部分检查在客户端完成，如确认Put实例是否为空，或是否指定了列。在这些情况下，客户端会抛出异常，并将错误的Put留在客户端缓冲区不做处理。
5. 使用put列表时，用户无法控制服务端执行put的顺序，如果要保证写入顺序，最坏的情况时减少每一批量处理操作数，并显示刷写缓冲区。

# checkAndPut
1. 当读取数据同时要修改数据时，使用checkAndPut方法来保证操作的原子性。   
    boolean checkAndPut(byte[] row, byte[] family, byte[] qualifier, byte[] value, Put put) throws IOException
2. 有一种特别的检查通过checkAndPut()调用完成，即只有在另一个值不存在的情况下，才执行修改。只需要将参数value的值设置为null即可，只要指定列不存在，就可以执行修改操作。***这里是否涉及到对两行数据得操作？与第三条是否冲突？***
3. checkAndPut()只能检查和修改同一行数据，这个操作只提供同一行数据的原子性保证，检查和修改分别针对不同行数据时会抛出异常。

# get方法
1. 使用get列表时，如果某个get方法中含有错误的列族，会导致整个get操作终止，程序会抛出异常，没有返回值。


# delete方法
1. deleteColumn()方法如果带有时间戳参数，则会只删除匹配时间戳的给定列的指定版本，不存在则不删除；如果不带时间戳参数，则删除给定列的最新版本。
2. deleteColumns()方法如果带有时间戳参数，则从给定列中删除与给定时间戳相等或更旧的版本；若不带时间戳参数，则删除给定列的所有版本。
3. 如果不设置时间戳，服务器会强制检索服务端最新时间戳，这比执行一个具有明确时间戳的删除要慢。

# 使用batch()进行批处理
1. void batch(List<Row> actions, Object[] results) throws IOException, InterruptedException   
    Object[] batch(List<Row> actions) throws IOException, InterruptedException   
    当抛出异常时，方法1能够将成功操作的结果填入results中，方法2则不能，因为新结果数组返回之前，控制流已经中断。
2. 类Row为Put、Get、Delete类的父类，这意味着这些操作都可以以多态的形式组装在List<Row>中。
3. 不可以将针对同一行的Put和Delete操作放在一个batch中，处于性能考虑，操作的处理顺序并不同，会导致不可预料的结果。
4. 示例：  
``` 
    List<Row> batch = new ArrayList<>();

    Put put = new Put(ROW1);
    put.add(COLFAM1,QUAL1,VALUE1);
    batch.add(put);

    Get get = new Get(ROW2);
    get.add(COLFAM2,QUAL2,VALUE2);
    batch.add(get);

    Delete delete = new Delete(ROW3);
    delete.add(COLFAM3,QUAL3,VALUE3);
    batch.add(delete);

    Object[] results = new Obejct[batch.size()];
    try{
        table.batch(batch,results);
    } catch(Exception e){
        e.printStackTrace();
    }

    for(int i = 0; i < results; i++){
        System.out.println("Reslt[" + i + "]: " + result[i]);
    }
```
5. 使用batch时，Put不会被写入缓冲区，batch()请求是同步的，没有延迟或其他中间操作。

## Lock & Scan
### Lock
1. 行锁用于保证只有一个客户端能获取一行数据相应的锁，同时对其进行修改。
2. 应尽量避免使用行锁，以防形成死锁。如果必须使用，应当限制占用锁的时间。
3. 当一个锁被显示获取，其他想要对该行数据加锁的客户端将会等待，直到锁被释放或租期超时。后者用于防止错误进程无限期占用锁。
4. Get操作不需要设置锁。
5. 当put操作被发送到服务器，如果没有显式指定时间戳，服务端会为其设定一个，而不是数据被写入时生成。
### Scan
1. 使用HTable的getScanner()方法来获取一个scan实例。   
    ResultScanner getScanner(Scan scan) throws IOException   
    ResultScanner getScanner(byte[] family) throws IOException   
    ResultScanner getScanner(byte[] family, byte[] qualifier) throws IOException   
    后两个方法会隐式创建scan实例。
2. scan的构造器   
    Scan()   
    Scan(byte[] startRow, Filter filter)
    Scan(byte[] startRow)   
    Scan(byte[] startRow, byte[] stopRow)   
3. 创建Scan实例后，仍然可以为其添加限制条件。   
    Scan addFamily(byte[] family)   
    Scan addColumn(byte[] family, byte[] qualifier)   
    Scan setStartRow(byte[] startRow)   
    Scan setStopRow(byte[] stopRow)   
    Scan setFilter(Filter filter)   


## ResultScanner
1. 扫描操作不会通过一次RPC请求返回所有匹配的行，而是以行为单位返回。
2. ResultScanner把扫描操作转换为类似的get操作，将每行数据封装为Result实例返回，并将所有Result放入一个迭代器中。
```
Scan scan = new Scan();
ResultScanner scanner = table.getScanner(scan);
for(Result res:scanner){
    System.out.println(res);
}
scanner2.close;
```

3. 需要在使用完ResultScanner后调用close()以释放资源。

4. 为了减少RPC请求次数，可以设置扫描器缓存，使得一次RPC请求获得多行数据。可以通过HTable.setScannerCaching(int canching)设置表记别的缓存；或者修改hbase-site.xml的参数来设置全局默认配置。
5. 需要为少量RPC请求和客户端、服务端内存消耗找到平衡，过大的缓冲区会导致更长的next()等待时间，以及更大的堆内存占用。
6. 数据量特别大的行，可以使用批量来减小每次的数据量。
```
void setBatch(int batch)
int getBatch()
```
缓存是面向行一级的操作，批量是面对列一级的操作。批量可以让用户选择每一次ResultScanner实例的next()操作返回多少列。如果一行的列数超过批量中设置的值，则会分片。如16列，会5 5 5 1地返回数据。

# Chapter IV
## 过滤器
### 过滤器层次结构
1. 最底层是Filter接口和FilterBase抽象类，大部分实体过滤器直接继承自FilterBase，也有一些间接继承自该类。
2. 有一组特殊过滤器，继承自CompareFilter，需要用户提供至少两个特定参数。
### 比较运算符
继承自CompareFilter地过滤器比基类FilterBase多了一个compare()方法，需要传入参数定义比较操作的过程。常用的有：LESS(匹配小于设定值的值), LESS_OR_EQUAL(小于等于), EQUAL, NOT_EQUAL, GREATER_OR_EQUAL, GREATER, NO_OP(排除一切值)
### 比较器
1. CompareFilter需要的第二类类型为比较器，常用的有：BinaryComparator(使用Bytes.compareTo()比较当前值与阈值), BinaryPrefixComparator(从左端开始前缀匹配), NullComparator(不做匹配，只判断当前值是不是null) BitComparator(位运算), RegexStringComparator(正则匹配), SubstringComparator(判断阈值是否是表中数据的子数组))
2. 后三种比较器只能与EQUAL和NOT_EQUAL搭配使用。
### 比较过滤器
1. 行过滤器， 基于行键。
2. 列族过滤器，过滤列族。
3. 列名过滤器，过滤列名。
4. 值过滤器，过滤值。
### 专用过滤器
1. 单列值过滤器，用一列的值决定是否一行数据被过滤。
2. 单列排除过滤器
3. 前缀过滤器，与前缀匹配的行都会被返回到客户端。
4. 分页过滤器，对结果进行分页，指定pageSize参数，控制每页返回行数。
...
### 自定义过滤器
通过实现Filter接口或继承FilterBase类，来实现自定义过滤器。
## 计数器
HBase的一种机制可以将列作为计数器。客户端API提供了专门的方法来完成这种**读取并修改**(read-and-modify)操作，同时在单独一次哭护短调用过程中保证原子性。
## 协处理器
### 简介
协处理器允许用户在region server上执行region级别的操作，并可以使用与RDBMS中**触发器**类似的功能。
### Coprocessor类
所有协处理器的类都必须实现这个接口。它定义了协处理器的基本约定，并使得框架本身的管理变得容易。   
它提供两个被应用于框架的枚举类（Priority和State），Priority有两个取值，分别为SYSTEM(高优先级，最先被执行)和USER(按顺序执行)。
提供如下两个方法：
```
void start(CoprocessorEnvironment env) throws IOException
void stop(CoprocessorEnvironment env) throws IOException
```
这两个方法在协处理器开始和结束时被调用。在协处理器实例的生命周期中，这两个方法 会被框架隐式调用，处理过程中每一步都有一个状态（即上文所提State）。(UNINSTALLED/INSTALLED/STARTING/ACTIVE/STOPPING/STOPPED)   
CoprocessorEnvironment用来在协处理器的生命周期中保持其状态。   
CoprocessorHost类用于维护所有协处理器实例和它们专用的环境。   
### 协处理器加载
1. 从配置中加载：通过添加多条配置到hbase-site.xml来添加协处理器。
2. 从表描述符中加载：使用HTableDescriptor.setValue()方法定义协处理器，仅针对特定表的region，也仅能被特定region server使用。
### RegionObserver
RegionObserver是Coprocessor的子类，属于observer协处理器：当一个特定的region级别操作发生时，它们的钩子函数会触发。这样的操作被分为两类：region生命周期变化和客户端API调用。
1. region生命周期变化：这些observer可以与pending open、open和pending close状态通过钩子链接。每一个钩子都被隐式地调用。
2. 处理客户端API事件：所有的客户端API调用都显式地从客户端应用中传输到region服务器。用户可以在这些调用执行前或刚刚执行后拦截它们。例如：
```
void preGet(...)/ void postGet(...)
```
在客户端HTable.get()请求执行之前和之后调用。

### MasterObserver
MasterObserver也是Coprocessor的子类，用于处理master服务器的所有回调函数。MasterObeserver提供一系列回调函数，例如：
```
void preCreateTable(...)/ void postCreateTable(...)
```
在表创建前后被调用。
### endpoint

## HTablePool


