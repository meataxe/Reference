# Delete temp table if exists

IF OBJECT_ID('tempdb..#Results') IS NOT NULL DROP TABLE #Results
GO