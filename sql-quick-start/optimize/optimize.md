## Lab 5: Implement and Optimize Vector Indexes:


## Task 1: Create an in-memory neighbor graph vector index (docs_hnsw_idx) on doc_chunks

1. Determine the memory allocation in the vector memory area.

 ```
select CON_ID, POOL, ALLOC_BYTES/1024/1024 as ALLOC_BYTES_MB,
USED_BYTES/1024/1024 as USED_BYTES_MB
from V$VECTOR_MEMORY_POOL order by 1,2;
 ```

2. Run an approximate similarity search to identify, within your books, the first four most relevant chunks.

    ```
    SELECT doc_id, chunk_id, chunk_data
    FROM doc_chunks
    ORDER BY vector_distance(chunk_embedding , :query_vector, COSINE)
    FETCH APPROX FIRST 4 ROWS ONLY WITH TARGET ACCURACY 80;
    ```

    You can also add a WHERE clause to further filter your search, for instance if you only want to look at one particular book. 

       ```
        SELECT doc_id, chunk_id, chunk_data
        FROM doc_chunks
        WHERE doc_id=1
        ORDER BY vector_distance(chunk_embedding , :query_vector, COSINE)
        FETCH APPROX FIRST 4 ROWS ONLY WITH TARGET ACCURACY 80;
        ```


## Task 2: Configure index settings (e.g., accuracy) using CREATE VECTOR INDEX statement.

1. Use the EXPLAIN PLAN command to determine how the optimizer resolves this query.

See Optimizer Plans for Vector Indexes for more information about how the Oracle Database optimizer uses vector indexes to run your approximate similarity searches. 

    ```
    EXPLAIN PLAN FOR
    SELECT doc_id, chunk_id, chunk_data
    FROM doc_chunks
    ORDER BY vector_distance(chunk_embedding , :query_vector, COSINE)
    FETCH APPROX FIRST 4 ROWS ONLY WITH TARGET ACCURACY 80;
    
    select plan_table_output from table(dbms_xplan.display('plan_table',null,'all'));
    ```

    ```
    PLAN_TABLE_OUTPUT
    --------------------------------------------------------------------------------
    Plan hash value: 2946813851
    --------------------------------------------------------------------------------------------------------
    | Id  | Operation                      | Name          | Rows  | Bytes |TempSpc| Cost (%CPU)| Time     |
    --------------------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT               |               |     4 |   104 |       |   12083 (2)| 00:00:01 |
    |*  1 |  COUNT STOPKEY                 |               |       |       |       |            |          |
    |   2 |   VIEW                         |               |  5014 |   127K|       |   12083 (2)| 00:00:01 |
    |*  3 |    SORT ORDER BY STOPKEY       |               |  5014 |    19M|    39M|   12083 (2)| 00:00:01 |
    |   4 |     TABLE ACCESS BY INDEX ROWID| DOC_CHUNKS    |  5014 |    19M|       |    1    (0)| 00:00:01 |
    |   5 |      VECTOR INDEX HNSW SCAN    | DOCS_HNSW_IDX |  5014 |    19M|       |    1    (0)| 00:00:01 |
    --------------------------------------------------------------------------------------------------------
    ```


 
2. Determine your vector index performance for your approximate similarity searches.

    The index accuracy reporting feature allows you to determine the accuracy of your vector indexes. After a vector index is created, you may be interested to know how accurate your approximate vector searches are.

```
SET SERVEROUTPUT ON
declare 
    report varchar2(128);
begin 
    report := dbms_vector.index_accuracy_query(
        OWNER_NAME => 'VECTOR', 
        INDEX_NAME => 'DOCS_HNSW_IDX',
        qv => :query_vector,
        top_K => 10, 
        target_accuracy => 90 );
    dbms_output.put_line(report); 
end; 
/
```
## Acknowledgements
- **Author** - Ana Coman, Database Product Management, July 2024
- **Contributors** - Ana Coman, Database Product Management, July 2024
- **Last Updated By/Date** - July 2024