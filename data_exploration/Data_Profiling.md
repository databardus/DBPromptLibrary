# Goal: Find data you're looking for based on expected usage and data properties
# Assumptions: The database you are currently connected to has the columns you're looking for.

# Parameters, and examples of values
Business domain: <Domain Name, like Sales, Manufacturaing Operations, etc.>
Column Purpose: <Sales Report, Operational Status>
Data Types: <int, varchar, nvarchar>
Schema: <dbo, dim>
Expected in the following clauses: WHERE, JOIN, SELECT

I am looking for a column and can't find it by name.
I need help identifying possible columns that might be what I'm looking for.

List possible candidate columns, by table name and column name, using the following criteria:
- The columns must show up frequently in queriest submitted to the database.
- The columns must have the data types defined in this prompt.
- Conceptually, the column must support the purpose above.
- The column would often be found in the clauses specified in the parameters section
- This column is/is not expected to be nullable.
- Focus on columns found in the defined schema.

Places to look for context to identify candidates:
- Column and table names
- Query Store for column usage in query text (if enabled)
- Columns included in index definitions