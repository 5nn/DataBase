#Fmdb数据库
--

- 创建数据库

```
NSString *filePath = [Domains stringByAppendingPathComponent:@"FMDB.db"];
FMDatabase *database = [FMDatabase databaseWithPath:filePath];

```

- 打开数据库

```
[database open];
```

- 更新操作

```
[database executeUpdateWithFormat:@"insert into mytable(num,name,sex) values(%d,%@,%@);",0,@"liuting","m"];

- (BOOL)executeUpdate:(NSString *)sql,... ;无参数模式
```

- 常见语句

>NSString *sqlStr = @"create table mytable(num integer,name varchar(7),sex char(1),primary key(num));";
>
> [database executeUpdateWithFormat:@"insert into mytable(num,name,sex) values(%d,%@,%@);",0,@"liuting","m"];
>
>UPDATE mytable set name = 'Fred' WHERE num = '127' //更新具体一行
>UPDATE Person SET Address = 'Zhongshan 23', City = 'Nanjing'
WHERE LastName = 'Wilson' //更新一行多列
>
>delete from mytable;删掉表
>delete from 表名称 where 列名称 = 值
>
>select * from mytable;// select
>select distinct xxx FROM mytable;//去除重复
> select * from mytable WHERE num >= 12 and sex = "1";
> select num, sex from mytable order by num ;//order by num desc,逆序

>limit常用来做分页查询
>select * from mytable limit 5;限制前五行
>
>
>select * from mytable limit 5,10;  ->6-15行数据
>
>
>select * From mytable Where num>=(
>Select num From mytable limit 90000,1
>)limit 100;
>
>选90001-90100行高效语句


- 查询操作

```
- (FMResultSet *)executeQuery:(NSString*)sql, ... ;
- (FMResultSet *)executeQueryWithFormat:(NSString*)format, ... ;
```
=== 
查询代码示例

```
- (NSArray *)getResultFromDatabase{
    //执行查询SQL语句，返回查询结果
    FMResultSet *result = [_database executeQuery:@"select * from mytable"];
    NSMutableArray *array = [NSMutableArray array];
    //获取查询结果的下一个记录
    while ([result next]) {
        //根据字段名，获取记录的值，存储到字典中
        NSMutableDictionary *dict = [NSMutableDictionary dictionary];
        int num  = [result intForColumn:@"num"];
        NSString *name = [result stringForColumn:@"name"];
        NSString *sex  = [result stringForColumn:@"sex"];
        dict[@"num"] = @(num);
        dict[@"name"] = name;
        dict[@"sex"] = sex;
        //把字典添加进数组中
        [array addObject:dict];
    }
    return array;
}

```

- FMDatabase线程安全问题 -->多个线程同时使用一个FMDatabase实例

#FMDBQueue

- create

```
FMDatabaseQueue *queue = [FMDatabaseQueue databaseQueueWithPath:path];
```

- opation

```
[queue inDatabase:^(FMDatabase*db) {
    //FMDatabase数据库操作
}];
```

代码样例

```
    FMDatabaseQueue * queue = [FMDatabaseQueue databaseQueueWithPath:self.dbPath];
    dispatch_queue_t q1 = dispatch_queue_create("queue1", NULL);
    dispatch_queue_t q2 = dispatch_queue_create("queue2", NULL);
    
    dispatch_async(q1, ^{
        for (int i = 0; i < 100; ++i) {
            [queue inDatabase:^(FMDatabase *db) {
                NSString * sql = @"insert into user (name, password) values(?, ?) ";
                NSString * name = [NSString stringWithFormat:@"queue111 %d", i];
                BOOL res = [db executeUpdate:sql, name, @"boy"];
                if (!res) {
                    debugLog(@"error to add db data: %@", name);
                } else {
                    debugLog(@"succ to add db data: %@", name);
                }
            }];
        }
    });
    
    dispatch_async(q2, ^{
        for (int i = 0; i < 100; ++i) {
            [queue inDatabase:^(FMDatabase *db) {
                NSString * sql = @"insert into user (name, password) values(?, ?) ";
                NSString * name = [NSString stringWithFormat:@"queue222 %d", i];
                BOOL res = [db executeUpdate:sql, name, @"boy"];
                if (!res) {
                    debugLog(@"error to add db data: %@", name);
                } else {
                    debugLog(@"succ to add db data: %@", name);
                }
            }];
        }
    });
    
```

#事务

demo code 

```
//事务  批量处理
-(void)transaction {
    // 开启事务
    [self.database beginTransaction]; 
    BOOL isRollBack = NO; 
    @try { 
        for (int i = 0; i<500; i++) { 
            NSNumber *num = @(i+1); 
            NSString *name = [[NSString alloc] initWithFormat:@"student_%d",i]; 
            NSString *sex = (i%2==0)?@"f":@"m";
            NSString *sql = @"insert into mytable(num,name,sex) values(?,?,?);"; 
            BOOL result = [database executeUpdate:sql,num,name,sex]; 
            if ( !result ) { 
                NSLog(@"插入失败！");
                return; 
            } 
        } 
    } 
    @catch (NSException *exception) { 
        isRollBack = YES; 
        // 事务回退
        [self.database rollback]; 
    } 
    @finally { 
        if (!isRollBack) { 
            //事务提交
            [self.database commit]; 
        } 
    } 
}
```