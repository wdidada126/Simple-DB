# my
/Users/ibqo/Develop/git/github/java/Simple-DB

```shell
convert /Users/ibqo/Develop/git/github/java/Simple-DB/files/file.txt 5
convert D:\develops\git\github\java\Simple-DB2\files\file.txt 5
```


```
print D:\develops\git\github\java\Simple-DB2\files\file.dat 5
print /Users/ibqo/Develop/git/github/java/Simple-DB/files/file.dat 5
```


```
1	2	3	4	5
1	2	3	4	5
1	2	3	4	5
1	2	3	4	5	
```


Field, IntField,StringField, Type

## class dsigram
Insert
Operator
OpIterator
Permissions
HeapPage
HeapFile
BufferPool

Insert



Catalog单个表Database.getCatalog()检索全局目录
Database.getBufferPool()
一个HeapFile对象被组织成一组页面，每个页面由固定数量的字节组成，用于存储元组（由常量BufferPool.DEFAULT_PAGE_SIZE定义），包括一个头部。在SimpleDB中，数据库中的每个表都有一个HeapFile对象。HeapFile中的每个页面被组织为一组槽位，每个槽位可以容纳一个元组（SimpleDB中给定表的所有元组大小相同）。除了这些槽位，每个页面还有一个头部，由每个元组槽位一位的位图组成。如果对应于特定元组的位是1，则表示该元组有效；如果是0，则表示该元组无效（例如，已被删除或从未初始化。）HeapFile对象的页面类型为HeapPage，它实现了Page接口。页面存储在缓冲池中，但由HeapFile类读取和写入。
SeqScan此操作符按顺序扫描构造函数中tableid指定的表的所有页面中的所有元组。此操作符应通过DbFile.iterator()方法访问元组。

```java
        // 构造一个3列表模式
        Type types[] = new Type[]{ Type.INT_TYPE, Type.INT_TYPE, Type.INT_TYPE };
        String names[] = new String[]{ "field0", "field1", "field2" };
        TupleDesc descriptor = new TupleDesc(types, names);

        // 创建表，将其与some_data_file.dat关联
        // 并告诉目录此表的模式。
        HeapFile table1 = new HeapFile(new File("some_data_file.dat"), descriptor);
        Database.getCatalog().addTable(table1, "test");

        // 构造查询：我们使用一个简单的SeqScan，它通过其迭代器提供元组。
        TransactionId tid = new TransactionId();
        SeqScan f = new SeqScan(tid, table1.getId());

        try {
            // 并运行它
            f.open();
            while (f.hasNext()) {
                Tuple tup = f.next();
                System.out.println(tup);
            }
            f.close();
            Database.getBufferPool().transactionComplete(tid);
        } catch (Exception e) {
            System.out.println ("Exception : " + e);
        }
```

HeapFile 堆文件，有表数据，表名称，列名称，列描述
表数据 .dat文件
表名称 TupleDesc
