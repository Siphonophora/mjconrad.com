---
layout: single
classes: wide
title:  "System Versioning 1 - When is SysStart"
date:   2020-05-09 09:00:00 -0400
permalink: blog/system-versioning-1-when-is-sysstart
header:
  teaser: "/images/2020/system-versioning-1/whenissysstart_square.png"
---

System versioning was introduced in SQL Server 2016. Its an excellent option for logging changes to data in sensitive systems and as with all complex systems, there some of the details are both important and have interesting consequences. This is the first in a series focusing on the 'fine print' in System Versioning. We will start off with a question that I couldn't find answered in the Microsoft docs, and which got me thinking more about some of the hidden complexities in system versioning.

When, exactly, is SysStart? 

To be more precise, how are the generated period columns (usually SysStart and SysEnd or PeriodStart and PeriodEnd) populated? 

The SQL below can demonstrate what is happening under the hood. The idea is to perform inserts with varying types of transactions, with each insert separated by a one second delay. We use the high precision `SYSUTCDATETIME()` function to so that the `InsertTime` is directly comparable to the period columns.

``` sql
CREATE TABLE [ExampleTable]  
(   
     [ID] [int] IDENTITY(1,1) NOT NULL PRIMARY KEY  
   , [SysStartTime] [datetime2](7) GENERATED ALWAYS AS ROW START NOT NULL   
   , [SysEndTime] [datetime2](7) GENERATED ALWAYS AS ROW END NOT NULL   
   , [InsertTime] [datetime2](7) NOT NULL
   , [TransactionType] [varchar](20) NOT NULL
   , PERIOD FOR SYSTEM_TIME ([SysStartTime], [SysEndTime])   
)    
WITH ( SYSTEM_VERSIONING = ON (HISTORY_TABLE = [dbo].[ExampleTableHistory] , DATA_CONSISTENCY_CHECK = ON ));   
GO   



INSERT INTO [ExampleTable]  (InsertTime, TransactionType)
VALUES (SYSUTCDATETIME(), 'Implicit');  


WAITFOR DELAY '00:00:01';  
INSERT INTO [ExampleTable]  (InsertTime, TransactionType)
VALUES (SYSUTCDATETIME(), 'Implicit Multi Row')
     , (SYSUTCDATETIME(), 'Implicit Multi Row');  

Begin Tran
    WAITFOR DELAY '00:00:01';  
    INSERT INTO [ExampleTable]  (InsertTime, TransactionType)
    VALUES  (SYSUTCDATETIME(), 'Explicit');  
    
    Begin Tran
        WAITFOR DELAY '00:00:01';   
        INSERT INTO [dbo].[ExampleTable]  (InsertTime, TransactionType)
        VALUES  (SYSUTCDATETIME(), 'Explicit (Nested)');  

    Commit Tran
Commit Tran

Select 
   *
 , DateDiff(ms, SysStartTime , InsertTime) SysStartToInsert_ms 
From [dbo].[ExampleTable] for system_time all


ALTER TABLE [dbo].[ExampleTable] Set (SYSTEM_VERSIONING = OFF); 
Drop table [dbo].[ExampleTable]
Drop table [dbo].[ExampleTableHistory]
```

This is the produced output:

<img src="{{ site.url }}{{ site.baseurl }}/images/2020/system-versioning-1/blog1.png" width="1000" alt="">

Lets take this one insert at a time. 
* **Implicit** - The single insert uses an implicit transaction. Its SysStart is almost identical to the `InsertTime`, as expected. I ran this code a few times in order to generate output where they weren't exactly the same. This illustrates that even in the implicit transaction, the timestamp is generated before the row insert.
* **Implicit Multi Row** - The statement inserts multiple rows in a single statement. Both rows have identical `SysStartTime`. This would be true regardless of the number of rows.
* **Explicit** - In this case we open an explicit transaction, wait one second, and then perform the insert. Indeed, we can see the `SysStartToInsert_ms` indicates that `SysStartTime` happened when `Begin Tran` was called, not when the insert happened. 
* **Explicit Nested** - Finally, we can see that nesting in another transaction does not establish a new SysStartTime, which is now 2 seconds earlier than the actual insert. 

If we did some updates or deletes as well, we would see the same rule applies:

Period Columns are populated using a timestamp which is generated at the beginning of the outermost transaction.
{: .notice--info} 

This fact results in some of the trickier behavior which can come out of System Versioning. It might be ideal if the timestamp was generated at the close of the transaction, because that is the moment the results are real. However, this is likely impossible. As SQL Server works through a transaction, it has to write transaction logs as it progresses. So from the very beginning of the transaction, it needs to know what values will populate the period columns.

In upcoming posts we will dig more into the implications of this for interpreting the period columns and performing 'time travel' queries.
