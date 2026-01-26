# Goal: Identify if Foreign Key constraints 
# have been defined in this database for a given column.

## Parameters
- ColumnName: <Name of the column, like CustomerID>

Find all tables that reference the ColumnName listed above, but don't have a formal foreign key 
constraint defined. Show me the relationships and suggest FK constraints I should add.