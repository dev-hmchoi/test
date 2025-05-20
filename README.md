-- 기존 임시 테이블 제거 및 재생성
IF OBJECT_ID('tempdb..#ToArchive') IS NOT NULL DROP TABLE #ToArchive;

CREATE TABLE #ToArchive (
    emp_id INT,
    emp_name NVARCHAR(100),
    dept_id INT,
    hire_date DATE
);

-- Oracle 쿼리 및 OPENQUERY 실행용 쿼리 구성
DECLARE @remote_select_sql NVARCHAR(MAX);
DECLARE @select_sql NVARCHAR(MAX);

SET @remote_select_sql = '
    SELECT emp_id, emp_name, dept_id, hire_date
    FROM employees
    WHERE dept_id = 10
      AND hire_date < TO_DATE(''2023-01-01'', ''YYYY-MM-DD'')
';

SET @select_sql = '
    INSERT INTO #ToArchive (emp_id, emp_name, dept_id, hire_date)
    SELECT * FROM OPENQUERY(ORACLE_LINKED_SERVER, ''' + @remote_select_sql + ''')
';

-- 실행
EXEC sp_executesql @select_sql;

-- 처리할 데이터가 있을 경우에만 커서 실행
IF EXISTS (SELECT 1 FROM #ToArchive)
BEGIN
    DECLARE @emp_id INT, @emp_name NVARCHAR(100), @dept_id INT, @hire_date DATE;
    DECLARE @remote_delete_sql NVARCHAR(MAX);
    DECLARE @delete_sql NVARCHAR(MAX);

    DECLARE archive_cursor CURSOR FOR
        SELECT emp_id, emp_name, dept_id, hire_date FROM #ToArchive;

    OPEN archive_cursor;
    FETCH NEXT FROM archive_cursor INTO @emp_id, @emp_name, @dept_id, @hire_date;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        BEGIN TRY
            BEGIN TRANSACTION;

            -- 기존에 존재하는 경우 UPDATE, 없으면 INSERT
            IF EXISTS (SELECT 1 FROM dbo.EmployeeArchive WHERE emp_id = @emp_id)
            BEGIN
                UPDATE dbo.EmployeeArchive
                SET emp_name = @emp_name,
                    dept_id = @dept_id,
                    hire_date = @hire_date
                WHERE emp_id = @emp_id;
            END
            ELSE
            BEGIN
                INSERT INTO dbo.EmployeeArchive (emp_id, emp_name, dept_id, hire_date)
                VALUES (@emp_id, @emp_name, @dept_id, @hire_date);
            END

            -- 아카이브에 존재하면 Oracle에서 DELETE, 없으면 오류 발생
            IF EXISTS (SELECT 1 FROM dbo.EmployeeArchive WHERE emp_id = @emp_id)
            BEGIN
                SET @remote_delete_sql = 
                    'DELETE FROM employees WHERE emp_id = ' + CAST(@emp_id AS VARCHAR);

                SET @delete_sql = 
                    'DELETE FROM OPENQUERY(ORACLE_LINKED_SERVER, ''' + @remote_delete_sql + ''')';

                EXEC sp_executesql @delete_sql;
            END
            ELSE
            BEGIN
                THROW 51000, '아카이브 실패: EmployeeArchive 테이블에 데이터가 없습니다.', 1;
            END

            COMMIT TRANSACTION;
        END TRY
        BEGIN CATCH
            ROLLBACK TRANSACTION;
            PRINT '에러 발생: ' + ERROR_MESSAGE();
        END CATCH

        FETCH NEXT FROM archive_cursor INTO @emp_id, @emp_name, @dept_id, @hire_date;
    END

    CLOSE archive_cursor;
    DEALLOCATE archive_cursor;
END
ELSE
BEGIN
    PRINT '처리할 데이터가 없습니다.';
END

---------------------------------
-- SQL Server 변수 정의
DECLARE @dept_id INT = 10;
DECLARE @hire_date_limit DATE = '2023-01-01';
DECLARE @emp_name_filter NVARCHAR(100) = 'KING';  -- 예시로 이름 필터도 추가

-- Oracle 쿼리 조립
DECLARE @remote_select_sql NVARCHAR(MAX);

SET @remote_select_sql = '
    SELECT emp_id, emp_name, dept_id, hire_date
    FROM employees
    WHERE dept_id = ' + CAST(@dept_id AS VARCHAR) + '
      AND hire_date < TO_DATE(''' + CONVERT(VARCHAR, @hire_date_limit, 120) + ''', ''YYYY-MM-DD'') 
      AND emp_name = ''' + REPLACE(@emp_name_filter, '''', '''''') + '''';

======================================










-- 임시 테이블에 먼저 데이터 적재 (커서를 위한 자료)
IF OBJECT_ID('tempdb..#ToArchive') IS NOT NULL DROP TABLE #ToArchive;

SELECT *
INTO #ToArchive
FROM OPENQUERY(ORACLE_LINKED_SERVER, '
    SELECT emp_id, emp_name, dept_id, hire_date
    FROM employees
    WHERE dept_id = 10
      AND hire_date < TO_DATE(''2023-01-01'', ''YYYY-MM-DD'')
');

-- 커서 선언
DECLARE @emp_id INT, @emp_name NVARCHAR(100), @dept_id INT, @hire_date DATE;

DECLARE archive_cursor CURSOR FOR
SELECT emp_id, emp_name, dept_id, hire_date FROM #ToArchive;

OPEN archive_cursor;
FETCH NEXT FROM archive_cursor INTO @emp_id, @emp_name, @dept_id, @hire_date;

WHILE @@FETCH_STATUS = 0
BEGIN
    BEGIN TRY
        BEGIN TRAN;

        -- 1. 아카이브 수행
        INSERT INTO dbo.EmployeeArchive (emp_id, emp_name, dept_id, hire_date)
        VALUES (@emp_id, @emp_name, @dept_id, @hire_date);

        -- 2. 아카이브 성공 시 삭제 수행
        IF EXISTS (
            SELECT 1 FROM dbo.EmployeeArchive WHERE emp_id = @emp_id
        )
        BEGIN
            DECLARE @delete_sql NVARCHAR(MAX);

            -- 리모트 Oracle 삭제 쿼리 조립
            SET @delete_sql = '
                DELETE FROM OPENQUERY(ORACLE_LINKED_SERVER, ''
                    DELETE FROM employees 
                    WHERE emp_id = ' + CAST(@emp_id AS VARCHAR) + '
                '')';

            EXEC sp_executesql @delete_sql;
        END

        COMMIT;
    END TRY
    BEGIN CATCH
        ROLLBACK;
        PRINT '에러 발생: ' + ERROR_MESSAGE();
    END CATCH

    FETCH NEXT FROM archive_cursor INTO @emp_id, @emp_name, @dept_id, @hire_date;
END

CLOSE archive_cursor;
DEALLOCATE archive_cursor;

