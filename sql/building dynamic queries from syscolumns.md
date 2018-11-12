# Meta-sql: building dynamic queries from syscolumns

This shows how to build up a query for a single piece of text in any/some columns using syscolumns and for-xml to flatten the result…

        DECLARE @SearchString AS VARCHAR(100)
        SET @SearchString = '''UN-KNOWN'''

        DECLARE @TableName AS VARCHAR(100)
        SET @TableName = 'CEM_CemeteryWarrant'

        -- Query to select the full dataset
        SELECT 
               'SELECT * FROM ' + @TableName + ' WHERE ' +

               STUFF(
                      (SELECT 
                             ' OR ' + sys.columns.name + '=' + @SearchString 
                      FROM 
                             sys.columns INNER JOIN sys.tables 
                                   ON sys.tables.object_id = sys.columns.object_id
                      WHERE 
                             sys.tables.name = @TableName
                             and system_type_id in (167,231) --varchar,nvarchar
                      FOR XML PATH('')      
               ), 1, 4, '') AS query

        -- Query to identify the counts per column of interest
        SELECT 
               REPLACE(
                      (STUFF(
                             (SELECT 
                                   ' UNION SELECT ''' + sys.columns.name + ''' AS ColumnName, COUNT(*) AS Counts FROM ' + @TableName + ' WHERE ' + sys.columns.name + '=' + @SearchString + ' HAVING COUNT(*) > 0'
                             FROM 
                                   sys.columns INNER JOIN sys.tables 
                                          ON sys.tables.object_id = sys.columns.object_id
                             WHERE 
                                   sys.tables.name = @TableName
                                   and system_type_id in (167,231) --varchar,nvarchar
                             FOR XML PATH('')
                      ), 1, 7, '') 
               ), '&gt;', '>')



