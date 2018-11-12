# Example SQL search in all columns of a type

        --Pick varchars from tables with <1k rows (checking look-up tables)
        select 
          tt.name AS table_name,
          c.name AS column_name,

          'union select ''[' + tt.name + '].[' + c.NAME + ']'' AS source, [' + c.name + '] AS data FROM ' + tt.name + ' WHERE (SELECT COUNT(*) FROM ' + tt.name + ') < 1000' AS query,

          column_id,
          t.name AS type_name,
          c.max_length,
          c.user_type_id,
          CASE WHEN d.definition IS NOT NULL AND d.definition = '(char(160))' THEN CHAR(160)
                WHEN d.definition IS NOT NULL AND d.definition = '(0)' THEN '0'
                ELSE d.definition
          END AS sql_default_value,
          COALESCE(d.definition, '') AS sql_default_definition,
          CASE WHEN t.name = 'varchar' THEN t.name + '(' + CAST(c.max_length AS varchar(10)) + ')' ELSE t.name END AS declaration
        from 
          sys.tables tt
          INNER JOIN sys.all_columns c ON tt.object_id = c.object_id
          left outer join sys.types t on c.user_type_id = t.user_type_id 
          left outer join sys.default_constraints d on c.default_object_id = d.object_id
        where 
          t.name LIKE '%varchar'  

        -- create searchable temp table
        IF object_id('tempdb..#source') IS NOT NULL
        BEGIN
           DROP TABLE #source
        END

        CREATE TABLE #source
        (
          source varchar(500),
          data varchar(MAX)
        )

        -- insert data, in blocks of 256 tables only (paste from the query col from 1st table)
        INSERT INTO #source
        (
          source,
          data
        )
        SELECT
          source,
          data
        FROM
        (
        -- union stuff goes here
        ) d

        -- analysis
        select 
          * 
        FROM 
          #source 
        WHERE
          COALESCE(data, '') <> ''
          AND data LIKE '%graf%-%'
        ORDER BY
          data

