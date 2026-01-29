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

Take a look at dbo.usp_ProcessEmployeeData, that loads data from dbo.Employees into stg.Employees and then into dim.Employees. I want to create a new stored procedure that loads data from dbo.Invoices into stg.Invoices and then into fact.Invoices. Check if the tables exist, and if they don't, provide queries to create those tables. Add a surrogate key to the fact table while maintaining the natural key. If the dbo table has columns with foreign key relationships, replace ID in the column name with Key in the final table.  

Take a look at dbo.usp_ProcessEmployeeData as the pattern stored procedure, that loads data from dbo.Employees into stg.Employees and then into dim.Employees. Using the same pattern, create a new stored procedure that loads data from dbo.Invoices into stg.Invoices and then into fact.Invoices. fact.Invoices is a fact table, so any logic that implements SCD Type 2 evolution for dim.Employees can be removed. When the pattern stored procedure replaces the natural ID from the source table and replaces it with a surrogate key using a subquery lookup, do the same in the procedure you generate if you find any columns that have foreign key relationships. When defining the subquery, check if the dimension table being looked up to has properties suggesting it is SCD Type 2 (i.e. EffectiveStartDate or EffectiveEndDate), and if so, be sure your lookup find the correct version of the dimension record using InvoiceDate. Also do not insert values into identity columns.

Take a look at dbo.usp_ProcessEmployeeData, that loads data from dbo.Employees into stg.Employees and then into dim.Employees. Using the same pattern, create a new stored procedure that loads data from dbo.Invoices into stg.Invoices and then into fact.Invoices. fact.Invoices is a fact table, so any logic that implements SCD Type 2 evolution for dim.Employees can be removed. Check if stg.Invoices and fact.Invoices exist, and if they don't, provide queries to create those tables. Also check for when the initial stored procedure replaces the natural ID from the source table and replaces it with a surrogate key, and do the same in the procedure you generate. Whatever queries you generate, go ahead and execute them against the database. If they generate errors, tell me what errors you encounter and see what you can do to resolve the errors. Make sure to check the actual name of the ErrorLog table in the database. Also do not insert values into identity columns.

## Customer example
## Table Creation
Take a look at dbo.usp_ProcessEmployeeData, that loads data from dbo.Employees into stg.Employees and then into dim.Employees. I want to create a new stored procedure that loads data from dbo.Customers into stg.Customers and then into dim.Customers. dim.Customers will be a SCD Type 2 dimension, so add in effective, expiration, and iscurrent columns as in dim.Employees. Check if all of these tables exist, and if they don't, provide queries to create those tables. If the dbo table has columns with foreign key relationships, replace ID in the column name with Key in the final table and allow columns in the staging table to support a mapping function from natural to surrogate key.

Take a look at dbo.usp_ProcessEmployeeData as the pattern stored procedure, that loads data from dbo.Employees into stg.Employees and then into dim.Employees. Using the same pattern, create a new stored procedure that loads data from dbo.Customers into stg.Customers and then into dim.Customers. Make sure you check the table definitions themselves so you are querying columns that exist. dim.Customers is a SCD Type 2 dimension table, so any logic that implements SCD Type 2 evolution for dim.Employees can be replicated. When the pattern stored procedure replaces the natural ID from the source table and replaces it with a surrogate key, do the same in the procedure you generate if you find any columns that have foreign key relationships. Also do not insert values into identity columns.

## New Invoice Example
Take a look at Employee Data, that loads data from dbo.Employees into stg.Employees and then into dim.Employees. I want to create new tables to support loading Invoice Data from dbo.Invoices into stg.Invoices and then into fact.Invoices. Check if the tables exist, and if they don't, provide queries to create those tables. Use only tables listed in this prompt as an example. Add a surrogate key to the fact table while maintaining the natural key. If the dbo table has columns with foreign key relationships, replace ID in the column name with Key in the final table.  Assume that the lookup operation for surrogate keys will occur in the staging table, so add columns there as necessary.
