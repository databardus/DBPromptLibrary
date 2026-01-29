# Goal: ETL processes often follow patterns that can be replicated for new data sources.
# Assumptions: An existing stored procedure exists that loads data.
# Also assumes that example processing tables exist in the database.

## Parameters
- Source Table Name (with schema): <dbo.Invoices>
- Staging Table Name (with schema): <stg.Invoices>
- Final Table Name (with schema): <fact.Invoices>
- Template Stored Procedure Name (with schema): <dbo.usp_ProcessSalesTransactions>
- Metadata Columns to Maintain: <CreatedDate, LastModifiedDate, ETLBatchID, ErrorFlag, ErrorMessage, InsertedDate, UpdatedDate>

## Build Tables
Take a look at the tables references in the template stored procedure, that loads data from source tables into staging tables 
and then into a final. Check if the corresponding tables exist to replicate this architecture for
the given source table, and if they don't, create those tables. 
Add a surrogate key to the final table while maintaining the natural key. 
If the source table has columns with foreign key relationships, replace ID in 
the column name with Key in the final table.
Only include metadata columns (listed in parameters above) that exist in the tables used in the template stored procedure.

## Build Stored Procedure
Take a look at the template stored procedure. Using the same pattern, create a new stored procedure 
that loads data from the source table into the final table. 
The final table is a 
fact/dimension table. Incorporate SCD Type X logic. 
Also check for when the template stored procedure replaces the natural ID from the source table and replaces 
it with a surrogate key, and do the same in the procedure you generate. 
Check the source table for columns with foreign key relationships and perform lookups as necessary. 
When defining the subquery, check if the dimension table being looked up to has properties suggesting it is SCD Type 2,
and look for the valid version of the dimension record using the appropriate date column.
When generating code that references a database object, check the database to make sure the object exists. 
Replicate any error logging logic exactly from the template stored procedure, making sure to check the actual name of the ErrorLog table in the database.
Replicate any batch processing logic exactly from the template stored procedure, making sure to check the actual name of the BatchLog table in the database.
Replicate any Try/Catch logic exactly from the template stored procedure.
