-- ================================================
-- Template generated from Template Explorer using:
-- Create Procedure (New Menu).SQL
--
-- Use the Specify Values for Template Parameters 
-- command (Ctrl-Shift-M) to fill in the parameter 
-- values below.
--
-- This block of comments will not be included in
-- the definition of the procedure.
-- ================================================
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Author,,Name>
-- Create date: <Create Date,,>
-- Description:	<Description,,>
-- =============================================
CREATE PROCEDURE <Procedure_Name, sysname, ProcedureName> 
	-- Add the parameters for the stored procedure here
	<@Param1, sysname, @p1> <Datatype_For_Param1, , int> = <Default_Value_For_Param1, , 0>, 
	<@Param2, sysname, @p2> <Datatype_For_Param2, , int> = <Default_Value_For_Param2, , 0>
AS
BEGIN

	-- SET NOCOUNT ON added to prevent extra result sets from
	declare aaa int;
	-- interfering with SELECT statements.


	SET NOCOUNT ON;



BEGIN TRY
    BEGIN TRAN;

	-- /* STEP 1. 마스터 테이블 처리
	-- 0. #t2501 존재 여부 확인
	IF NOT EXISTS (SELECT 1 FROM #t2501)
    BEGIN
        ROLLBACK;
        THROW 50001, '#t2501 실패 - 아카이브할 데이터가 존재하지 않습니다.', 1;
		--  or 로깅 표시 후 정상 종료
		/*
		PRINT '경고: #t2501에 아카이브할 데이터가 존재하지 않습니다.';
		-- 또는 로깅 테이블에 기록
		INSERT INTO error_log (error_code, message, created_at) VALUES ('W001', 't2501 insert 이후 데이터 없음', GETDATE());
		RETURN 0;
		*/
    END


    -- 1. t2501HS UPDATE
    UPDATE t2501HS -- pk 제외
    SET col1 = tmp.col1,
        col2 = tmp.col2
    FROM t2501HS t
    INNER JOIN #t2501 tmp ON t.pk = tmp.pk;

    IF @@ROWCOUNT = 0 -- 업데이트 된것이 없으면
    BEGIN
        -- 1.1 t2501HS INSERT
        INSERT INTO t2501HS (pk, col1) VALUES (@pk, '값');

        -- 1.2 t2501HS insert 재확인
        IF NOT EXISTS (SELECT 1 FROM t2501HS WHERE pk = @pk)
        BEGIN
            ROLLBACK;
            THROW 50001, 't2501HS INSERT 실패 - 데이터가 정상적으로 INSERT되지 않았습니다.', 1;
        END
    END
	-- end STEP 1. 마스터 테이블 처리 */

	-- /* STEP 2. 자식테이블 아카이브 처리
    -- 2. t2501HS가 존재 -> t2502HS UPDATE
	IF EXISTS (SELECT 1 FROM #t2502)
	BEGIN
		-- #t2502 총10개, update->3, insert->7를 고려

		-- 1. UPDATE: 복합키 조건으로 대상 행만 수정
		UPDATE T
		SET T.col1 = S.col1,
			T.col2 = S.col2
		FROM t2502HS AS T
		INNER JOIN #t2502 AS S ON T.pk1 = S.pk1 AND T.pk2 = S.pk2;

		-- 2. INSERT: 조인이 되지 않은 행만 선별하여 INSERT
		INSERT INTO t2502HS (pk1, pk2, col1, col2)
		SELECT S.pk1, S.pk2, S.col1, S.col2
		FROM #t2502 AS S
		LEFT JOIN t2502HS AS T ON S.pk1 = T.pk1 AND S.pk2 = T.pk2
		WHERE T.pk1 IS NULL;


		-- 3. 검증: insert 이후에도 존재하지 않으면 롤백 및 오류 발생
		IF EXISTS (
			SELECT 1
			FROM #t2502 AS S
			LEFT JOIN t2502HS AS T
				ON S.pk1 = T.pk1 AND S.pk2 = T.pk2
			WHERE T.pk1 IS NULL
		)
		BEGIN
			ROLLBACK;
			THROW 50001, 't2502 INSERT 실패 - 일부 데이터가 존재하지 않습니다.', 1;
		END
	END
	-- NOT EXISTS #t2502 -> 데이타가 없을 수도 있음 -> pass


    -- 2. t2501HS가 존재 -> t2505HS UPDATE
	IF EXISTS (SELECT 1 FROM #t2505)
	BEGIN
	END
	-- 2. t2501HS가 존재 -> t2505HS UPDATE

	-- end STEP 2. 자식테이블 아카이브 처리 */







    COMMIT;
END TRY
BEGIN CATCH
    IF @@TRANCOUNT > 0 ROLLBACK;
    PRINT ERROR_MESSAGE();
END CATCH






END
GO




-------------------------------------
네, 제공해주신 SQL Server 프로시저 템플릿을 보다 안정적이고 효율적으로 개선하는 방법을 제안해 드립니다. 현재 코드의 좋은 부분은 유지하되, 성능을 저해할 수 있는 요소들을 제거하고, 더 나은 예외 처리 및 로깅 방식을 추가했습니다.
개선된 SQL Server 프로시저 템플릿
아래는 개선된 프로시저 코드입니다. 각 개선 사항에 대한 자세한 설명은 코드 다음에 이어집니다.
CREATE PROCEDURE dbo.YourProcedureName
    @InputParam1 INT = 0,
    @InputParam2 NVARCHAR(100),
    @OutputParam1 INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    SET XACT_ABORT ON; -- 트랜잭션 내에서 런타임 오류 발생 시 자동 롤백

    -- ▒▒ 변수 선언 ▒▒
    DECLARE @LocalVar1 INT;
    DECLARE @LocalVar2 INT = 100;
    DECLARE @SumResult INT;
    DECLARE @ErrorMessage NVARCHAR(4000);
    DECLARE @ErrorSeverity INT;
    DECLARE @ErrorState INT;
    DECLARE @IsSuccess BIT = 1;

    -- ▒▒ 초기 값 설정 및 유효성 검사 (필요 시) ▒▒
    SET @LocalVar1 = @InputParam1 * 10;

    -- @InputParam2가 NULL이거나 빈 문자열인 경우 처리
    IF @InputParam2 IS NULL OR @InputParam2 = ''
    BEGIN
        SET @OutputParam1 = -1;
        -- 좀 더 상세한 오류 메시지 또는 로깅
        RAISERROR(N'InputParam2는 NULL이거나 빈 문자열일 수 없습니다.', 16, 1);
        RETURN -1;
    END

    BEGIN TRY
        BEGIN TRANSACTION;

        SET @SumResult = @LocalVar1 + @LocalVar2;

        -- ▒▒ Step 1. 임시 테이블 생성 및 데이터 로딩 (ROW_NUMBER 포함) ▒▒
        -- IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL DROP TABLE #TargetList; -- 불필요, WITH 절 사용 시
        -- CTE 또는 테이블 변수를 사용하여 임시 테이블 사용 최소화
        -- 또는 필요한 경우 #TargetList를 프로시저 시작 전에 확실히 DROP하고 CREATE

        -- 대량의 데이터를 처리해야 한다면 #TargetList를 생성하는 것이 유리
        IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL DROP TABLE #TargetList;
        SELECT
            ROW_NUMBER() OVER (ORDER BY SomeColumn) AS RowNum,
            SomeColumn
        INTO #TargetList
        FROM YourTable
        WHERE SomeCondition = @InputParam2;

        -- ▒▒ Step 2. 데이터 유무 확인 및 처리 ▒▒
        IF NOT EXISTS (SELECT 1 FROM #TargetList)
        BEGIN
            PRINT N'처리할 데이터가 없습니다.';
            SET @OutputParam1 = 0;
            COMMIT TRANSACTION; -- 처리할 데이터가 없으므로 트랜잭션 커밋
            RETURN 0;
        END

        -- ▒▒ Step 3. 커서 대신 Set-based 또는 WHILE 루프 최적화 ▒▒
        -- 일반적으로 커서는 성능 저하의 주범입니다. 가능하면 Set-based(집합 기반) 연산으로 대체하세요.
        -- 예: UPDATE YourTable SET Column = NewValue WHERE SomeColumn IN (SELECT SomeColumn FROM #TargetList);

        -- 만약 루프가 불가피하다면, 최적화된 WHILE 루프나 APPLY 연산을 고려하세요.
        -- 이 템플릿에서는 기존 WHILE 루프를 유지하되, 잠재적 문제점을 설명합니다.
        DECLARE @CursorRowNum INT = 1;
        DECLARE @TotalRowCount INT = (SELECT COUNT(*) FROM #TargetList);
        DECLARE @CurrentSomeColumn NVARCHAR(100); -- 현재 처리할 SomeColumn 값

        WHILE @CursorRowNum <= @TotalRowCount
        BEGIN
            SELECT @CurrentSomeColumn = SomeColumn
            FROM #TargetList
            WHERE RowNum = @CursorRowNum;

            -- NULL 데이터 체크는 WHERE 절에서 필터링하는 것이 더 효율적일 수 있습니다.
            -- 또는 서브 프로시저에서 NULL 허용 여부를 명확히 하는 것이 좋습니다.
            IF @CurrentSomeColumn IS NULL
            BEGIN
                -- 오류 처리 대신 경고 또는 로깅 후 계속 진행할지 결정해야 합니다.
                -- 여기서는 심각한 오류로 간주하고 중단합니다.
                SET @IsSuccess = 0;
                RAISERROR(N'#TargetList에 NULL 데이터가 발견되었습니다. 처리 중단.', 16, 1);
                BREAK; -- 루프 중단
            END

            -- ★ 서브 프로시저 호출 (예시) ★
            -- 서브 프로시저가 트랜잭션을 자체적으로 처리하는 경우, 외부 트랜잭션과 충돌할 수 있으니 주의
            EXEC dbo.YourSubProcedure
                @Param1 = @CurrentSomeColumn,
                @Param2 = @InputParam1;

            -- 진행률 출력은 개발/디버깅용으로만 사용하고, 프로덕션에서는 성능에 영향을 줄 수 있으므로 제거 고려
            PRINT N'진행률: ' + CAST(@CursorRowNum AS NVARCHAR(20)) + N' / ' + CAST(@TotalRowCount AS NVARCHAR(20));

            SET @CursorRowNum += 1;

            -- 비정상적인 루프 횟수 체크는 유효하지만, 보통 데이터 유효성 검사 또는 로직 문제 해결이 우선
            IF @CursorRowNum > 100000 -- 적절한 최대 루프 횟수 설정
            BEGIN
                SET @IsSuccess = 0;
                RAISERROR(N'루프 횟수가 비정상적으로 많습니다. 처리 중단합니다.', 16, 1);
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
            -- @IsSuccess가 0이면 이미 RAISERROR로 CATCH 블록으로 이동했거나,
            -- 트랜잭션이 아직 열려있을 경우에만 ROLLBACK 처리
            IF XACT_STATE() <> 0
                ROLLBACK TRANSACTION;
            SET @OutputParam1 = -1;
            -- 이미 RAISERROR로 오류를 발생시켰으므로 여기서는 추가 RETURN -1만 합니다.
            RETURN -1;
        END

    END TRY
    BEGIN CATCH
        -- 오류 발생 시 트랜잭션 롤백
        IF XACT_STATE() <> 0
            ROLLBACK TRANSACTION;

        SELECT
            @ErrorMessage = ERROR_MESSAGE(),
            @ErrorSeverity = ERROR_SEVERITY(),
            @ErrorState = ERROR_STATE();

        -- 오류 로깅 (예: 오류 테이블에 삽입)
        -- INSERT INTO ErrorLogTable (ProcedureName, ErrorCode, ErrorMessage, ErrorSeverity, ErrorState, LogDate)
        -- VALUES (OBJECT_NAME(@@PROCID), ERROR_NUMBER(), @ErrorMessage, @ErrorSeverity, @ErrorState, GETDATE());

        -- 클라이언트에게 오류 메시지 전달 (선택 사항)
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);

        SET @OutputParam1 = -1;
        RETURN -1; -- 오류 발생 시 -1 반환
    END CATCH

    -- ▒▒ Step 5. 임시 테이블 정리 ▒▒
    IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL
        DROP TABLE #TargetList;

    -- ▒▒ Step 6. 리턴 코드 ▒▒
    RETURN 0; -- 성공 시 0 반환
END;
GO

주요 개선 사항 및 설명
1. SET XACT_ABORT ON; 추가
 * 목적: 트랜잭션 내에서 런타임 오류(예: 제약 조건 위반)가 발생할 경우, SQL Server가 자동으로 현재 트랜잭션을 롤백하고 프로시저 실행을 중단하게 합니다. 이는 트랜잭션의 일관성을 보장하고, CATCH 블록에서 XACT_STATE()를 확인하는 번거로움을 줄여줍니다.
2. 초기 파라미터 유효성 검사
 * @InputParam2와 같이 필수적인 입력 파라미터에 대해 NULL 또는 빈 문자열 체크를 추가했습니다. 프로시저 실행 초기에 문제를 감지하여 불필요한 연산을 방지합니다.
 * RAISERROR를 사용하여 명확한 오류 메시지를 발생시키고 프로시저를 즉시 종료합니다.
3. PRINT 문구 개선
 * PRINT 문구에 N 접두사를 사용하여 유니코드 문자열임을 명시했습니다. (예: PRINT N'처리할 데이터가 없습니다.';)
 * CAST(@CursorRowNum AS NVARCHAR) 대신 CAST(@CursorRowNum AS NVARCHAR(20))처럼 명시적인 길이를 지정하는 것이 좋습니다.
4. WHILE 루프 최적화 및 CURSOR 대안 (Set-based Operation)
 * 가장 중요한 개선 사항입니다. SQL Server에서 WHILE 루프 (특히 ROW_NUMBER를 기반으로 한 루프)는 **커서(Cursor)**와 유사하게 동작하며, 대량의 데이터를 처리할 때 심각한 성능 저하를 일으킬 수 있습니다.
 * 권장 사항:
   * 가능한 한 Set-based (집합 기반) 연산으로 로직을 변경하세요. 즉, UPDATE, INSERT, DELETE 문에 JOIN, WHERE, CTE 등을 활용하여 한 번의 쿼리로 여러 행을 처리하는 방식입니다.
   * 예를 들어, dbo.YourSubProcedure에서 수행하는 작업이 여러 행에 대해 동일한 로직을 적용하는 것이라면, 이를 한 번의 UPDATE 또는 INSERT...SELECT 문으로 대체하는 것을 적극적으로 고려해야 합니다.
 * 루프가 불가피한 경우:
   * 루프 내에서 매번 SELECT @CursorValue = SomeColumn FROM #TargetList WHERE RowNum = @CursorRowNum;을 수행하는 것은 비효율적입니다. SELECT 한 번으로 필요한 모든 데이터를 가져와 처리하거나, APPLY 연산자를 사용하여 외부 쿼리의 결과에 따라 함수나 서브쿼리를 실행하는 방식을 고려할 수 있습니다.
   * 현재 템플릿에서는 WHILE 루프를 유지했지만, 성능 개선을 위해서는 이 부분을 가장 먼저 검토해야 합니다.
5. 트랜잭션 처리 강화
 * COMMIT TRANSACTION 또는 ROLLBACK TRANSACTION 전에 IF XACT_STATE() <> 0을 통해 현재 트랜잭션이 유효한 상태인지 확인하는 것은 좋은 습관입니다.
 * CATCH 블록에서 ERROR_MESSAGE(), ERROR_SEVERITY(), ERROR_STATE()를 명확히 가져와서 RAISERROR로 다시 발생시키거나 오류 로깅을 수행할 수 있도록 했습니다. 이는 문제 진단에 큰 도움이 됩니다.
6. 임시 테이블 (#TargetList) 관리
 * IF OBJECT_ID('tempdb..#TargetList') IS NOT NULL DROP TABLE #TargetList; 구문은 프로시저 시작 시와 종료 시에 모두 사용하는 것이 좋습니다. 시작 시에 두는 이유는 이전 실행에서 비정상적으로 종료되어 임시 테이블이 남아있을 경우를 대비합니다.
 * 만약 #TargetList에 저장되는 데이터의 양이 적고, 루프가 매우 짧게 돈다면 **테이블 변수 (DECLARE @TargetList TABLE (...))**를 사용하는 것을 고려할 수 있습니다. 테이블 변수는 트랜잭션에 포함되지 않고 세션이 종료되면 자동 삭제되므로 DROP TABLE이 필요 없지만, 통계 정보가 없어 대량 데이터 처리에는 불리합니다.
7. 로깅 및 오류 메시지
 * 단순 PRINT 대신 오류 로깅 테이블을 사용하는 것을 강력히 권장합니다. 프로시저 실행 중 발생한 오류나 중요한 이벤트들을 데이터베이스에 기록함으로써, 운영 중 발생한 문제들을 추적하고 분석할 수 있습니다.
 * RAISERROR는 단순히 PRINT하는 것보다 더 강력한 오류 처리 메커니즘을 제공합니다. @ErrorSeverity를 16 이상으로 설정하면 클라이언트 애플리케이션에서 이를 오류로 인식하고 처리할 수 있습니다.
이 개선된 템플릿은 SQL Server 프로시저의 견고성과 성능을 향상시키는 데 중점을 둡니다. 특히 루프 처리 부분은 데이터 처리량에 따라 성능에 가장 큰 영향을 미치므로, 실제 비즈니스 로직에 맞춰 Set-based 방식으로 변경하는 것을 최우선으로 고려하시길 바랍니다.


----------------

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

