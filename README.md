CREATE PROCEDURE dbo.YourProcedureName
    @InputParam1 INT = 0,
    @InputParam2 NVARCHAR(100),
    @OutputParam1 INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    -- ▒▒ 변수 선언 ▒▒
    DECLARE @LocalVar1 INT = 0;
    DECLARE @LocalVar2 INT = 100;
    DECLARE @SumResult INT;
    DECLARE @ErrorCode INT = 0;
    DECLARE @ErrorMessage NVARCHAR(4000);
    DECLARE @IsSuccess BIT = 1;

    DECLARE @CursorRowNum INT = 1;         -- 커서용 순번 (ROW_NUMBER 기준)
    DECLARE @CursorValue NVARCHAR(100);
    DECLARE @TotalRowCount INT = 0;

    -- ▒▒ Step 1. 임시 테이블 생성 (ROW_NUMBER 포함) ▒▒
    IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL DROP TABLE #TargetList;

    SELECT 
        ROW_NUMBER() OVER (ORDER BY SomeColumn) AS RowNum,
        SomeColumn
    INTO #TargetList
    FROM YourTable
    WHERE SomeCondition = @InputParam2;

    -- ▒▒ Step 2. 총 건수 계산 ▒▒
    SELECT @TotalRowCount = COUNT(*) FROM #TargetList;

    IF @TotalRowCount = 0
    BEGIN
        PRINT '처리할 데이터가 없습니다.';
        SET @OutputParam1 = 0;
        RETURN 0;
    END

    BEGIN TRY
        BEGIN TRANSACTION;

        SET @LocalVar1 = @InputParam1 * 10;
        SET @SumResult = @LocalVar1 + @LocalVar2;

        -- ▒▒ Step 3. 루프 처리 (ROW_NUMBER 기반) ▒▒
        WHILE @CursorRowNum <= @TotalRowCount
        BEGIN
            SELECT @CursorValue = SomeColumn
            FROM #TargetList
            WHERE RowNum = @CursorRowNum;

            IF @CursorValue IS NULL
            BEGIN
                PRINT 'NULL 데이터 발견. 중단합니다.';
                SET @IsSuccess = 0;
                BREAK;
            END

            -- ★ 서브 프로시저 호출 (예시) ★
            EXEC dbo.YourSubProcedure
                @Param1 = @CursorValue,
                @Param2 = @InputParam1;

            PRINT '진행률: ' + CAST(@CursorRowNum AS NVARCHAR) + ' / ' + CAST(@TotalRowCount AS NVARCHAR);

            SET @CursorRowNum += 1;

            IF @CursorRowNum > 100000
            BEGIN
                RAISERROR('루프 횟수가 비정상적으로 많습니다. 중단합니다.', 16, 1);
                SET @IsSuccess = 0;
                BREAK;
            END
        END

        -- ▒▒ Step 4. 성공 여부에 따른 트랜잭션 처리 ▒▒
        IF @IsSuccess = 1
        BEGIN
            COMMIT TRANSACTION;
            SET @OutputParam1 = @SumResult;
        END
        ELSE
        BEGIN
            IF XACT_STATE() <> 0
                ROLLBACK TRANSACTION;
            SET @OutputParam1 = -1;
            RETURN -1;
        END
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        SET @ErrorCode = ERROR_NUMBER();
        SET @ErrorMessage = ERROR_MESSAGE();
        PRINT '에러 발생: ' + CAST(@ErrorCode AS NVARCHAR) + ' - ' + @ErrorMessage;

        SET @OutputParam1 = -1;
        RETURN -1;
    END CATCH

    -- ▒▒ Step 5. 임시 테이블 정리 ▒▒
    IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL
        DROP TABLE #TargetList;

    -- ▒▒ Step 6. 리턴 코드 ▒▒
    RETURN 0;
END;
GO


CREATE PROCEDURE dbo.YourProcedureName
    @InputParam1 INT = 0,
    @InputParam2 NVARCHAR(100),
    @OutputParam1 INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    -- ▒▒ 변수 선언 ▒▒
    DECLARE @LocalVar1 INT = 0;
    DECLARE @LocalVar2 INT = 100;
    DECLARE @SumResult INT;
    DECLARE @ErrorCode INT = 0;
    DECLARE @ErrorMessage NVARCHAR(4000);
    DECLARE @IsSuccess BIT = 1;

    DECLARE @CursorRowNum INT = 1;         -- 커서용 순번 (ROW_NUMBER 기준)
    DECLARE @CursorValue NVARCHAR(100);
    DECLARE @TotalRowCount INT = 0;

    -- ▒▒ Step 1. 임시 테이블 생성 (ROW_NUMBER 포함) ▒▒
    IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL DROP TABLE #TargetList;

    SELECT 
        ROW_NUMBER() OVER (ORDER BY SomeColumn) AS RowNum,
        SomeColumn
    INTO #TargetList
    FROM YourTable
    WHERE SomeCondition = @InputParam2;

    -- ▒▒ Step 2. 총 건수 계산 ▒▒
    SELECT @TotalRowCount = COUNT(*) FROM #TargetList;

    IF @TotalRowCount = 0
    BEGIN
        PRINT '처리할 데이터가 없습니다.';
        SET @OutputParam1 = 0;
        RETURN 0;
    END

    BEGIN TRY
        BEGIN TRANSACTION;

        SET @LocalVar1 = @InputParam1 * 10;
        SET @SumResult = @LocalVar1 + @LocalVar2;

        -- ▒▒ Step 3. 루프 처리 (ROW_NUMBER 기반) ▒▒
        WHILE @CursorRowNum <= @TotalRowCount
        BEGIN
            SELECT @CursorValue = SomeColumn
            FROM #TargetList
            WHERE RowNum = @CursorRowNum;

            IF @CursorValue IS NULL
            BEGIN
                PRINT 'NULL 데이터 발견. 중단합니다.';
                SET @IsSuccess = 0;
                BREAK;
            END

            -- ★ 서브 프로시저 호출 (예시) ★
            EXEC dbo.YourSubProcedure
                @Param1 = @CursorValue,
                @Param2 = @InputParam1;

            PRINT '진행률: ' + CAST(@CursorRowNum AS NVARCHAR) + ' / ' + CAST(@TotalRowCount AS NVARCHAR);

            SET @CursorRowNum += 1;

            IF @CursorRowNum > 100000
            BEGIN
                RAISERROR('루프 횟수가 비정상적으로 많습니다. 중단합니다.', 16, 1);
                SET @IsSuccess = 0;
                BREAK;
            END
        END

        -- ▒▒ Step 4. 성공 여부에 따른 트랜잭션 처리 ▒▒
        IF @IsSuccess = 1
        BEGIN
            COMMIT TRANSACTION;
            SET @OutputParam1 = @SumResult;
        END
        ELSE
        BEGIN
            IF XACT_STATE() <> 0
                ROLLBACK TRANSACTION;
            SET @OutputParam1 = -1;
            RETURN -1;
        END
    END TRY
    BEGIN CATCH
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        SET @ErrorCode = ERROR_NUMBER();
        SET @ErrorMessage = ERROR_MESSAGE();
        PRINT '에러 발생: ' + CAST(@ErrorCode AS NVARCHAR) + ' - ' + @ErrorMessage;

        SET @OutputParam1 = -1;
        RETURN -1;
    END CATCH

    -- ▒▒ Step 5. 임시 테이블 정리 ▒▒
    IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL
        DROP TABLE #TargetList;

    -- ▒▒ Step 6. 리턴 코드 ▒▒
    RETURN 0;
END;
GO

-------------------
BEGIN TRY
    BEGIN TRAN;

    -- 임시 테이블 생성
    CREATE TABLE #TempResult (
        Id INT,
        Name NVARCHAR(100),
        Status NVARCHAR(20),
        Amount DECIMAL(18, 2),
        CreatedDate DATETIME
    );

    -- 파라미터 선언
    DECLARE 
        @Status NVARCHAR(20) = 'ACTIVE',
        @MinAmount DECIMAL(18, 2) = 1000.00,
        @StartDate DATE = '2024-01-01',
        @SQL NVARCHAR(MAX);

    -- 동적 SQL (파라미터 바인딩 방식)
    SET @SQL = N'
        SELECT 
            Id,
            Name,
            Status,
            Amount,
            CreatedDate
        FROM RemoteDB.dbo.Orders
        WHERE Status = @P_Status
          AND Amount >= @P_MinAmount
          AND CreatedDate >= @P_StartDate
    ';

    -- 결과 삽입
    INSERT INTO #TempResult (Id, Name, Status, Amount, CreatedDate)
    EXEC sp_executesql 
        @SQL,
        N'@P_Status NVARCHAR(20), @P_MinAmount DECIMAL(18,2), @P_StartDate DATE',
        @P_Status = @Status,
        @P_MinAmount = @MinAmount,
        @P_StartDate = @StartDate
    AT [RemoteServer];

    -- 결과 사용
    SELECT * FROM #TempResult;

    -- 커밋
    COMMIT;
END TRY
BEGIN CATCH
    PRINT '에러 발생: ' + ERROR_MESSAGE();
    IF @@TRANCOUNT > 0
        ROLLBACK;
END CATCH;

-- 임시 테이블 정리
IF OBJECT_ID('tempdb..#TempResult') IS NOT NULL
    DROP TABLE #TempResult;






BEGIN TRY
    BEGIN TRAN;

    -- 임시 테이블 생성 (결과에 맞게 컬럼 정의)
    CREATE TABLE #TempResult (
        Id INT,
        Name NVARCHAR(100),
        Status NVARCHAR(20),
        Amount DECIMAL(18, 2),
        CreatedDate DATETIME
    );

    -- 조건 정의 (예: 문자열, 숫자, 일자 포함)
    DECLARE 
        @Status NVARCHAR(20) = 'ACTIVE',
        @MinAmount DECIMAL(18, 2) = 1000.00,
        @StartDate DATE = '2024-01-01',
        @SQL NVARCHAR(MAX);

    -- 동적 SQL 생성
    SET @SQL = N'
        SELECT 
            Id,
            Name,
            Status,
            Amount,
            CreatedDate
        FROM RemoteDB.dbo.Orders
        WHERE Status = ''' + @Status + ''' 
          AND Amount >= ' + CAST(@MinAmount AS NVARCHAR) + '
          AND CreatedDate >= ''' + CONVERT(NVARCHAR, @StartDate, 120) + '''';

    -- 결과 삽입
    INSERT INTO #TempResult (Id, Name, Status, Amount, CreatedDate)
    EXEC (@SQL) AT [RemoteServer];

    -- 결과 사용 예시
    SELECT * FROM #TempResult;

    -- 성공 시 커밋
    COMMIT;

END TRY
BEGIN CATCH
    -- 오류 정보 출력
    PRINT '에러 발생: ' + ERROR_MESSAGE();

    -- 롤백
    IF @@TRANCOUNT > 0
        ROLLBACK;

END CATCH

-- 임시 테이블 정리
IF OBJECT_ID('tempdb..#TempResult') IS NOT NULL
    DROP TABLE #TempResult;




-- 기존 임시 테이블 제거 및 재생성
IF OBJECT_ID('tempdb..#ToArchive') IS NOT NULL DROP TABLE #ToArchive;

    hire_date 

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

