# Example SQL search in all columns #2

Purpose: To search all columns of all tables for a given search string

        DECLARE @SearchStr NVARCHAR(100)
        SET @SearchStr = '158521'

        DECLARE @Results TABLE 
          (
            ColumnName NVARCHAR(370),
            ColumnValue NVARCHAR(3630)
          )

        SET NOCOUNT ON

        DECLARE
          @TableName NVARCHAR(256),
          @ColumnName NVARCHAR(128),
          @SearchStr2 NVARCHAR(110)
        SET @TableName = ''
        SET @SearchStr2 = QUOTENAME('%' + @SearchStr + '%', '''')

        WHILE @TableName IS NOT NULL 
          BEGIN
            SET @ColumnName = ''
            SET @TableName = (
                                SELECT
                                MIN(QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME))
                                FROM
                                INFORMATION_SCHEMA.TABLES
                                WHERE
                                TABLE_TYPE = 'BASE TABLE'
                                AND QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME) > @TableName
                                AND OBJECTPROPERTY(OBJECT_ID(QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME)), 'IsMSShipped') = 0
                              )

            WHILE ( @TableName IS NOT NULL )
              AND ( @ColumnName IS NOT NULL ) 
              BEGIN
                SET @ColumnName = (
                                    SELECT
                                      MIN(QUOTENAME(COLUMN_NAME))
                                    FROM
                                      INFORMATION_SCHEMA.COLUMNS
                                    WHERE
                                      TABLE_SCHEMA = PARSENAME(@TableName, 2)
                                      AND TABLE_NAME = PARSENAME(@TableName, 1)
                                      AND QUOTENAME(COLUMN_NAME) > @ColumnName
                                      --AND DATA_TYPE = 'int'
                                  )

                IF @ColumnName IS NOT NULL 
                  BEGIN
                    INSERT  INTO @Results
                            EXEC (
                                    'SELECT ''' + @TableName + '.' + @ColumnName + ''', LEFT(CONVERT(varchar(max), ' + @ColumnName + '), 3630) 
                    FROM ' + @TableName + ' (NOLOCK) ' + ' WHERE CONVERT(varchar(max), ' + @ColumnName + ') LIKE ' + @SearchStr2
                                )
                  END
              END 
          END

        SELECT DISTINCT
          ColumnName,
          ColumnValue,
          'SELECT * FROM ' + PARSENAME(ColumnName, 3) + '.' + PARSENAME(ColumnName, 2) + ' WHERE ' + PARSENAME(ColumnName, 1) + ' = ''' + ColumnValue + ''''
        FROM
          @Results


