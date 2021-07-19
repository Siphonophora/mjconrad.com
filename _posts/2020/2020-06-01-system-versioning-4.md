---
layout: single
classes: wide
title:  "System Versioning 4 - Avoid No-Op Updates"
date:   2020-06-01 09:00:00 -0400
permalink: blog/system-versioning-4-avoid-no-op-updates
---

When updating a row of data, SQL Server does not check if the new values and the old values are the same before performing the update. This often isn't a concern, but when using System Versioning no-op updates should be avoided. 

We can see this clearly with a quick example.

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

    INSERT INTO [ExampleTable]  (Comment)
    VALUES ('ValueA')
         , ('ValueB');  

    Update [ExampleTable] Set Comment = 'ValueA'

Select * From [ExampleTable] For System_Time All Order By ID, SysStartTime

ALTER TABLE [dbo].[ExampleTable] Set (SYSTEM_VERSIONING = OFF); 
Drop table [dbo].[ExampleTable]
Drop table [dbo].[ExampleTableHistory]
```

Results:

![](/images/2020/system-versioning-4/noopupdate.png)

Here we can see that the unchanged row was updated. We now have an *unnecessary* row which was generated in our audit trail. This behavior needs to be considered when designing updates for system versioned tables, as normal patterns for ETL or ORM updates could generate large volumes of History table data. In most cases, avoid these unnecessary updates is straight forward, but this is an area where simply enabling system versioning in an existing system could get you into trouble.