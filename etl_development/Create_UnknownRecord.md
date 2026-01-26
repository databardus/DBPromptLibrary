# Goal: Design the prompt to consistently define unknown records on tables.
# Assumptions: There is an existing table that contains the metadata columns, but not necessary.
# Look for < or > characters to identify places where the prompt should be modified to suit your needs.

Look at the schema of <identify table>. Write a script to add an Unknown record to the table, 
populating all of the columns with appropriate 'Unknown' values. 
Integer fields should have a value of <value>.
String fields should have a value of <value>.
Date fields should have a value of <value>.
NULLable columns <can/cannot> remain null. 
The record should have a value of -1 in the key column. 
Make sure to change the Identity Insert property so you can insert the record with the ID.