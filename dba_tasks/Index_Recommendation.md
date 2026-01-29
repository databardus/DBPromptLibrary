# Goal: Analyze a specific table and provide index recommendations

## Parameters
- TableName: <Name of the table, like SalesOrders>
- SchemaName: <Name of the schema, like dbo>

### Prompt  
I would like to analyze and get index recommendations for the given table in the given schema. 
Please include the following in your analysis:  
- The table structure (column details, data types, constraints).  
- The volume of data currently in the table.  
- Any missing index recommendations for the table.  
- Any usage or query patterns for this table from the Query Store or other available statistics.  
- Suggestions to improve performance via indexing based on identified patterns.  

Provide any recommended scripts to create the suggested indexes, as well as explanations on how they might improve query performance.
