# Goal: Create data in a table that requires a foreign key relationship.
# Assumptions: The target table and foreign key lookup table exist.


Write a script that, Given a table with existing data, <source_table_name>, populates the associated foreign key column of <target_table_name>, <target_column>, with available values from the table. 
Randomly select which ID ends up on which Invoice record using a cursor to iterate.