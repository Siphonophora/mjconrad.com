---
layout: single
classes: wide
title:  "System Versioning 2 - Concurrent Transactions"
date:   2020-05-18 09:00:00 -0400
permalink: blog/system-versioning-2-concurrent-transactions
---

In the second installment of this series we are going to look at how concurrent transactions behave differently with System Versioned tables, compared to standard SQL tables.

As a simplified example, we will take one long running transaction which updates some rows and another short transaction which updates a single record while the long running transaction is open. In the examples, one file is provided for each session executing against the database. The code is run in order noted in the comments.

#### Non-Overlapping Data

In the first example the two transactions do not overlap in the rows they update.

``` sql
-- ================ Run First ================
    CREATE TABLE [ExampleTable]  
    (   
         [ID] [int] IDENTITY(1,1) NOT NULL PRIMARY KEY  
       , [SysStartTime] [datetime2](7) GENERATED ALWAYS AS ROW START NOT NULL   
       , [SysEndTime] [datetime2](7) GENERATED ALWAYS AS ROW END NOT NULL   
       , [InsertTime] [datetime2](7) NOT NULL
       , [Comment] [varchar](50) NOT NULL
       , PERIOD FOR SYSTEM_TIME ([SysStartTime], [SysEndTime])   
    )    
    WITH ( SYSTEM_VERSIONING = ON (HISTORY_TABLE = [dbo].[ExampleTableHistory], 
                                   DATA_CONSISTENCY_CHECK = ON ));   
    GO   

    INSERT INTO [ExampleTable]  (InsertTime, Comment)
    VALUES (SYSUTCDATETIME(), 'Create 3 Rows')
         , (SYSUTCDATETIME(), 'Create 3 Rows')
         , (SYSUTCDATETIME(), 'Create 3 Rows');  

-- ================ Run Third ================
    Update ExampleTable 
    Set InsertTime = SYSUTCDATETIME()
      , Comment = 'Update 2 - Short Transaction'
    Where ID = 2
```
``` sql
-- ================ Run Second ================
    Begin Tran

    Update ExampleTable 
    Set InsertTime = SYSUTCDATETIME()
      , Comment = 'Update 1 - LongRunning Transaction - Start'
    Where ID = 1

-- ================ Run Fourth ================
    Update ExampleTable 
    Set InsertTime = SYSUTCDATETIME()
      , Comment = 'Update 3 - LongRunning Transaction - End'
    Where ID = 3

    Commit Tran

    Select * 
    From ExampleTable 
    For System_Time All
    Order by SysStartTime

    
    ALTER TABLE [dbo].[ExampleTable] Set (SYSTEM_VERSIONING = OFF); 
    Drop table [dbo].[ExampleTable]
    Drop table [dbo].[ExampleTableHistory]
```

As we can see, all the updates have occurred, but the order they happened, according to `SysStartTime` is obscured somewhat. Update 1 and 3 look like they happen at the same time, and before update 2. In most situations this won't create any problems, but its important to keep in mind if your workload has longer transactions.

![](/images/2020/system-versioning-2/ConcurrentNonOverlappingOutput.PNG)

#### Overlapping Data

This is essentially the same situation as above, except that both transactions are updating the same row. 

``` sql
-- ================ Run First ================
    CREATE TABLE [ExampleTable]  
    (   
         [ID] [int] IDENTITY(1,1) NOT NULL PRIMARY KEY  
       , [SysStartTime] [datetime2](7) GENERATED ALWAYS AS ROW START NOT NULL   
       , [SysEndTime] [datetime2](7) GENERATED ALWAYS AS ROW END NOT NULL   
       , [InsertTime] [datetime2](7) NOT NULL
       , [Comment] [varchar](50) NOT NULL
       , PERIOD FOR SYSTEM_TIME ([SysStartTime], [SysEndTime])   
    )    
    WITH ( SYSTEM_VERSIONING = ON (HISTORY_TABLE = [dbo].[ExampleTableHistory],
                                   DATA_CONSISTENCY_CHECK = ON ));   
    GO   

    INSERT INTO [ExampleTable]  (InsertTime, Comment)
    VALUES (SYSUTCDATETIME(), 'Create 1 Row');  

-- ================ Run Third ================
    Update ExampleTable 
    Set InsertTime = SYSUTCDATETIME()
      , Comment = 'Update 1 - Short Transaction'
    Where ID = 1
```
``` sql
-- ================ Run Second ================
    Begin Tran

-- ================ Run Fourth ================

    Update ExampleTable 
    Set InsertTime = SYSUTCDATETIME()
        , Comment = 'Update 2 - LongRunning Transaction - End'
    Where ID = 1

    
    ALTER TABLE [dbo].[ExampleTable] Set (SYSTEM_VERSIONING = OFF); 
    Drop table [dbo].[ExampleTable]
    Drop table [dbo].[ExampleTableHistory]
```

In this case, we get an error when session 2 tries to update the row.

><span style="color:red">Msg 13535, Level 16, State 0, Line 6
Data modification failed on system-versioned table 'BlogDb.dbo.ExampleTable' because transaction time was earlier than period start time for affected records.</span>
The statement has been terminated.

Without this error, we would end up end up with a real problem in the history table. The `Update 1 - Short Transaction` row would have a period with a negative time span. Its easy to imagine all the problems this would cause with the 'time travel' queries, so it makes sense this would generate an error. 

Its worth thinking about the impact of this restriction on a production system, particularly if you are looking at the possibility of converting existing tables to use system versioning. In a standard table, the exact same sequence of events would complete without any errors. This acts a lot like an optimistic concurrency check. However, this could be an issue that will happen randomly and be difficult to debug. 
* If the long running transaction had taken done its update before the short running tran, the latter would have had to wait for the transaction to finish and wouldn't raise any error. Therefore, there is some randomness in when this error would occur. 
* Unlike the deadlock victim error, this error can't provide any indication of who made the change that caused the long running transaction to fail.

The only changed behavior here is for **concurrent updates** as shown above. Concurrent inserts will work without issue, and concurrent transactions trying to delete the same record would behave the same as with standard tables.