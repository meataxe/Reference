# Generate EF class stuff from SQL via query

        SELECT
          '_namespace Root.Thing.Orm.Views_{_  using System.Collections.Generic;_  using System.ComponentModel.DataAnnotations;__  // ReSharper disable InconsistentNaming_  public class '
          + name + '_  {_  }_}___' AS class_part,
          'public DbSet<' + name + '> ' + name + ' { get; set; }' AS entity_part,
          'modelBuilder.Entity<' + name + '>().ToTable("' + name + '");' AS model_builder_part
        FROM
          sys.objects
        WHERE
          type = 'V'
        ORDER BY
          name;



        -- props part
        SELECT
          v.name,
          c.name,
          t.name,
          'public ' + CASE
                        WHEN t.name IN ( 'varchar', 'nvarchar', 'nchar', 'char' ) THEN
                          'string'
                        WHEN t.name = 'datetime' THEN
                          'DateTime'
                        WHEN t.name = 'bit' THEN
                          'bool'
                        ELSE
                          t.name
                      END + IIF(c.is_nullable = 1 AND t.name NOT IN ( 'varchar', 'nvarchar', 'nchar', 'char' ), '?', '') + ' '
          + c.name + ' { get; set; }',
          LOWER(LEFT(c.name, 1)) + RIGHT(c.name, LEN(c.name) - 1) + ': { type: "'
          + CASE
              WHEN t.name IN ( 'varchar', 'nvarchar', 'nchar', 'char' ) THEN
                'string'
              WHEN t.name IN ( 'int', 'bigint' ) THEN
                'number'
              WHEN t.name = 'datetime' THEN
                'date'
              WHEN t.name = 'bit' THEN
                'boolean'
              ELSE
                t.name
            END + '" }, ' AS KendoSchemaFields
        FROM
          sys.columns c
          INNER JOIN sys.views v
          ON c.object_id = v.object_id

          INNER JOIN sys.types t
          ON c.system_type_id = t.user_type_id
        ORDER BY
          v.name,
          c.name;

        -- PK’s
        SELECT
          o.name AS [Table],
          c.name AS PK
        FROM
          sys.objects o
          INNER JOIN sys.indexes i
          ON o.object_id = i.object_id
             AND i.is_primary_key = 1

          INNER JOIN sys.index_columns ic
          ON i.object_id = ic.object_id
             AND i.index_id = ic.index_id

          INNER JOIN sys.columns c
          ON ic.object_id = c.object_id
             AND ic.column_id = c.column_id
        WHERE
          o.type = 'U'
        ORDER BY
          o.name;

        -- relationships - doesn't pick up child-via-pk relationships
        SELECT
          FK = FK.TABLE_NAME + '.' + CU.COLUMN_NAME,
          FK_Attribute = '[ForeignKey("' + PK.TABLE_NAME + '")]',
          FK_Instance = 'public virtual ' + PK.TABLE_NAME + ' ' + PK.TABLE_NAME + ' { get; set; }',
          PK = PK.TABLE_NAME, -- + '.' + PT.COLUMN_NAME,
                              --Constraint_Name = C.CONSTRAINT_NAME,
          PK_Collection = 'public virtual ICollection<' + FK.TABLE_NAME + '> ' + FK.TABLE_NAME
                          + CASE
                              WHEN LEFT(REVERSE(FK.TABLE_NAME), 1) = 'S' THEN
                                ''
                              ELSE
                                's'
                            END + ' { get; set; }'
        FROM
          INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS C
          INNER JOIN INFORMATION_SCHEMA.TABLE_CONSTRAINTS FK
          ON C.CONSTRAINT_NAME = FK.CONSTRAINT_NAME

          INNER JOIN INFORMATION_SCHEMA.TABLE_CONSTRAINTS PK
          ON C.UNIQUE_CONSTRAINT_NAME = PK.CONSTRAINT_NAME

          INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE CU
          ON C.CONSTRAINT_NAME = CU.CONSTRAINT_NAME

          INNER JOIN
            (
              SELECT
                i1.TABLE_NAME,
                i2.COLUMN_NAME
              FROM
                INFORMATION_SCHEMA.TABLE_CONSTRAINTS i1
                INNER JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE i2
                ON i1.CONSTRAINT_NAME = i2.CONSTRAINT_NAME
              WHERE
                i1.CONSTRAINT_TYPE = 'PRIMARY KEY'
            ) PT
          ON PT.TABLE_NAME = PK.TABLE_NAME
        ORDER BY
          1,
          2,
          3,
          4;
