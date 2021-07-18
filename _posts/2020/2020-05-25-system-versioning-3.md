---
layout: single
classes: wide
title:  "System Versioning 3 - Inter-Transaction Changes Are Not Recorded"
date:   2020-05-25 09:00:00 -0400
permalink: blog/system-versioning-3-inter-transaction-changes-are-not-recorded
---

In this post we will take another look at how transactions impact system versioned tables. There is exception to the rule that a system versioned table will record *all* changes to records: within a transaction. 

Below we have some sample code that creates two records. Within the same transaction, one record is updated an the other is deleted. 

``` sql
CREATE TABLE [ExampleTable]  
(   
      [ID] [int] IDENTITY(1,1) NOT NULL PRIMARY KEY  
    , [SysStartTime] [datetime2](7) GENERATED ALWAYS AS ROW START NOT NULL   
    , [SysEndTime] [datetime2](7) GENERATED ALWAYS AS ROW END NOT NULL   
    , [Comment] [varchar](50) NOT NULL
    , PERIOD FOR SYSTEM_TIME ([SysStartTime], [SysEndTime])   
)    
WITH ( SYSTEM_VERSIONING = ON (HISTORY_TABLE = [dbo].[ExampleTableHistory],
                               DATA_CONSISTENCY_CHECK = ON ));   
GO   

BEGIN TRAN
    INSERT INTO [ExampleTable]  (Comment)
    VALUES ('CreateThenUpdate')
         , ('CreateThenDelete');  

    Update [ExampleTable] Set Comment = 'Updated' Where Comment = 'CreateThenUpdate'
    Delete From [ExampleTable] Where Comment = 'CreateThenDelete'

COMMIT TRAN

Select * From [ExampleTable] For System_Time All

ALTER TABLE [dbo].[ExampleTable] Set (SYSTEM_VERSIONING = OFF); 
Drop table [dbo].[ExampleTable]
Drop table [dbo].[ExampleTableHistory]
```

The output shows that only the *final state* of the transaction was recorded. We see the 'updated' value as if it was the value that was inserted. The deleted never really existed. 

![](/images/2020/system-versioning-3/With-Transaction.png)

Just for contrast, the results below are what we would get if the transaction was not used. 

![](/images/2020/system-versioning-3/Without-Transaction.png)

One way to think about what is happening here is this. Sql Server will only ever move a row to the history table on update or delete, if that row existed before the transaction began. Additionally, it will only create one new version of a row within a transaction.