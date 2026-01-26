# Goal: Design the prompt to consistently define metadata columns on tables.
# Assumptions: There is an existing table that contains the metadata columns.

Given a source table <source_metadata_table>, 
a target table <target_metadata_table>, 
and an integer <column_number>,
add metadata columns to the target table so they match the metadata column on the source table.
Metadata Columns are defined as the last integer number of columns of the source table.
Check whether the columns exist before adding them to the target table.
Be sure to replicate data types when defining the columns on the target table.