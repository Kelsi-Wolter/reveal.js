In April 2024 our team ran into this exact problem on a large partitioned-table, how did we solve it?
---
## Background Context
- New (replacement) table to house successful shipping label requests
- Database interactions:
    - Writes: once for each successful request
    - Read & Update: cancellation requests
    - Reads: reprint requests
- Table is partitioned by list - using "created date"


<img src="images/v2/label_partitions2.png">
---
## A Performance Issue Appears
- Problem arose after migrating the cancellation functionality to the new table
- Cancellation requests were taking 5x longer than they used to
- Upon further investigation, we found that there was a missing index from the previous table
- Solution: add index for tracking number and location ID to the new table
---
## Adding an index: Considerations
- P1 system - performance and downtime must always be considered when making database changes
- Adding indexes will lock tables as read-only while the index is being built
- Adds load to the server, causing slowness as well


- Using `CONCURRENTLY` helps to avoid the table lock, however...
```"...one limitation when creating new indexes on partitioned tables is that it is not possible to use the CONCURRENTLY qualifier, which could lead to long lock times."```
- Postgres suggests a process with a few more steps...
---
## Adding an index to a partitioned table
- From the docs:
    - Step 1: Add the index to the partitioned table *only*
  ``` 
    create index successful_label_cancel_idx 
    on only successful_label (tracking_number,location_id);
  ```
  - marks the index as `invalid`
  - this does not apply the index to existing partitions
  - this would not be applied to any future partitions


<img src="images/v2/table-idx.png">


- Step 2: Add the index to each partition using *CONCURRENTLY*
  ```
  CREATE INDEX CONCURRENTLY successful_label_20240417_cancel_idx
    ON successful_label_20240417 (tracking_number,location_id);
  ```


<img src="images/v2/partitions-idx.png">


- Step 3: Attach all partition indexes to the table index
  ```
  ALTER INDEX successful_label_cancel_idx
  ATTACH PARTITION successful_label_20240417_cancel_idx;
  ```
<img src="images/v2/valid-idx.png">


- once the index meets all 3 criteria, the index is marked as `valid`  
  ✅ index exists on the partitioned table  
  ✅ index exists on all partitions  
  ✅ partition indexes are attached to the table index  
- once the index is marked as valid, it will automatically be applied to future generated partitions 🎉
---
Executing the Steps in a Table Partitioned by List/Date  
<img src="images/partitions2.png"/>
- Table is partitioned by date
- Data in the past was less of a concern as we would not be writing new entries to those old partitions
- Main concern was to avoid locking the current partition


## Solution
- complete process over 2 days
- Day 1
  - create the index on the partitioned table `ONLY`
  - create the partition for the next day's date WITH the index


<img src="images/v2/step-1v2.png">


Day 2
- on the next day add the index to the older partitions using *CONCURRENTLY*


<img src="images/v2/step-2v2.png">


- attach all partition indexes to the table index
- parent index becomes valid!
<img src="images/v2/final-step2.png">
---
## The outcome
- no downtime!
- indexes were generated on all newly generated partitions!
- latency on cancellation requests improved from previous performance, now using the new partitioned table with correct indexes!
---
Thank you!