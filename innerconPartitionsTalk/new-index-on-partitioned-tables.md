Background
- Built a new table to house successful shipping label requests
- Database interactions:
    - Writes: once for each successful request
    - Reads: cancellation and reprint requests
- Table is partitioned by list - using "upload date"
---
Slowness
- Problem arose after migrating the cancellation functionality to the new table
- Cancellation requests were taking 5x longer than they used to
- Upon further investigation, we found that there was a missing index on tracking number
- Solution: add index for tracking number to new table
---
Adding an index
- P1 system - performance and downtime must always be considered when making database changes
- Adding indexes can make tables read-only while the index is being built


- Postgres has a feature called CONCURRENTLY that allows you to add an index without locking the table
- HOWEVER, Postgres does not support adding indexes concurrently to partitioned tables
- Additionally, adding an index to a partitioned table so that the index applies to newly generated partitions requires a few extra steps
---
Adding an index to a partitioned table
- From the docs:
    - Step 1: Add the index to the parent table only
  ``` 
    create index successful_label_cancel_idx 
    on only shipmentlabeldb.successful_label (tracking_number,location_id);
  ```
  - marks the index as "invalid"
  - this would not be applied to any future partitions
  - this does not apply the index to child partitions


- Step 2: Add the index to the child tables using *CONCURRENTLY*
  ```
  CREATE INDEX CONCURRENTLY successful_label_20240417_cancel_idx
    ON shipmentlabeldb.successful_label_20240417 (tracking_number,location_id);
  ```


- Step 3: Attach all child indexes to the parent index
  ```
  ALTER INDEX shipmentlabeldb.successful_label_cancel_idx
  ATTACH PARTITION shipmentlabeldb.successful_label_20240417_cancel_idx;
  ```


- once the index meets all 3 criteria, the index is marked as "valid"
    - [x] added to existing partitions
    - [x] child indexes are attached to the parent index
    - [x] AND the parent table index exists
- once the index is valid, it is automatically applied to future partitions
---
Our process
- Table is partitioned by date
- Data in the past was less of a concern as we would not be writing new entries to those old partitions
- Main concern was to avoid locking the current partition


Solution
- create a new partition for tomorrow WITH the index
- then on the next day add the index to the older partitions using *CONCURRENTLY*
- attach all partition indexes to the parent index
- parent index becomes valid!
---
The outcome
- milliseconds of downtime!
- indexes were generated on all new partitions!
- latency on cancellation requests improved from previous performance, now using the new paritioned table with correct indexes!
