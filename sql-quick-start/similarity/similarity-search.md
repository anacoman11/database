## Lab 4: Generate Query Vector for Similarity Search

## Introduction


## Task 1: For a similarity search you will need query vectors.

1. Here you enter your query text and generate an associated vector embedding
For example, you can use the following text: 'different methods of backup and recovery'. You use the VECTOR_EMBEDDING SQL function to generate the vector embeddings from the input text. The function takes an embedding model name and a text string to generate the corresponding vector.

In SQL*Plus, use the following code:


    ```
    ACCEPT text_input CHAR PROMPT 'Enter text: '
    VARIABLE text_variable VARCHAR2(1000)
    VARIABLE query_vector VECTOR
    BEGIN
    :text_variable := '&text_input';
    SELECT vector_embedding(doc_model using :text_variable as data) into :query_vector;
    END;
    /
    
    PRINT query_vector
    ```
    
    In SQLCL, use the following code:

        
    ```
    DEFINE text_input = '&text'
    
    SELECT '&text_input';
    
    VARIABLE text_variable VARCHAR2(1000)
    VARIABLE query_vector CLOB
    BEGIN
    :text_variable := '&text_input';
    SELECT vector_embedding(doc_model using :text_variable as data) into :query_vector;
    END;
    /
    
    PRINT query_vector
    ```

2. Run a similarity search to find, within your books, the first four most relevant chunks that talk about backup and recovery.

    Using the generated query vector, you search similar chunks in the DOC_CHUNKS table. For this, you use the VECTOR_DISTANCE SQL function and the FETCH SQL clause to retrieve the most similar chunks.


    ```
    SELECT doc_id, chunk_id, chunk_data
    FROM doc_chunks
    ORDER BY vector_distance(chunk_embedding , :query_vector, COSINE)
    FETCH FIRST 4 ROWS ONLY;
    ```


## Task 2: Print and verify the generated query vector

1. Use the EXPLAIN PLAN command to determine how the optimizer resolves this query.

    ```
    EXPLAIN PLAN FOR
    SELECT doc_id, chunk_id, chunk_data
    FROM doc_chunks
    ORDER BY vector_distance(chunk_embedding , :query_vector, COSINE)
    FETCH FIRST 4 ROWS ONLY;
    
    select plan_table_output from table(dbms_xplan.display('plan_table',null,'all'));
    ```

     ```
    PLAN_TABLE_OUTPUT
    ------------------------------------------------------------------------------
    Plan hash value: 1651750914
    -------------------------------------------------------------------------------------------------
    | Id  | Operation               | Name          | Rows  | Bytes |TempSpc| Cost (%CPU)| Time     |
    -------------------------------------------------------------------------------------------------
    |   0 | SELECT STATEMENT        |               |     4 |   104 |       |   549   (3)| 00:00:01 |
    |*  1 |  COUNT STOPKEY          |               |       |       |       |            |          |
    |   2 |   VIEW                  |               |  5014 |   127K|       |   549   (3)| 00:00:01 |
    |*  3 |    SORT ORDER BY STOPKEY|               |  5014 |   156K|   232K|   549   (3)| 00:00:01 |
    |   4 |     TABLE ACCESS FULL   | DOC_CHUNKS    |  5014 |   156K|       |   480   (3)| 00:00:01 |
    -------------------------------------------------------------------------------------------------
     ```

2. Run a multi-vector similarity search to find, within your books, the first four most relevant chunks in the first two most relevant books.

    Here you keep using the same query vector as previously use

    ```
    SELECT doc_id, chunk_id, chunk_data
    FROM doc_chunks
    ORDER BY vector_distance(chunk_embedding , :query_vector, COSINE)
    FETCH FIRST 2 PARTITIONS BY doc_id, 4 ROWS ONLY;
    ```

3. Create an In-Memory Neighbor Graph Vector Index on the vector embeddings that you created.

    ```
    create vector index docs_hnsw_idx on doc_chunks(chunk_embedding)
    organization inmemory neighbor graph
    distance COSINE
    with target accuracy 95;
    
    SELECT INDEX_NAME, INDEX_TYPE, INDEX_SUBTYPE
    FROM USER_INDEXES;
    INDEX_NAME     INDEX_TYPE  INDEX_SUBTYPE
    -------------- ----------- -----------------------------
    DOCS_HNSW_IDX  VECTOR      INMEMORY_NEIGHBOR_GRAPH_HNSW

    ...
    
    SELECT JSON_SERIALIZE(IDX_PARAMS returning varchar2 PRETTY)
    FROM VECSYS.VECTOR$INDEX where IDX_NAME = 'DOCS_HNSW_IDX';
    JSON_SERIALIZE(IDX_PARAMSRETURNINGVARCHAR2PRETTY)
    ________________________________________________________________
    {
    "type" : "HNSW",
    "num_neighbors" : 32,
    "efConstruction" : 300,
    "distance" : "COSINE",
    "accuracy" : 95,
    "vector_type" : "FLOAT32",
    "vector_dimension" : 384,
    "degree_of_parallelism" : 1,
    "pdb_id" : 3,
    "indexed_col" : "CHUNK_EMBEDDING"
    }
    ```



## Acknowledgements
- **Author** - Ana Coman, Database Product Management, July 2024
- **Contributors** - Ana Coman, Database Product Management, July 2024
- **Last Updated By/Date** - July 2024

