-- Begin Transaction for safety
BEGIN TRANSACTION;

-- Step 1: Process each request in SubjectRequest
DECLARE @StudentId VARCHAR(50);
DECLARE @RequestedSubjectId VARCHAR(50);
DECLARE @CurrentSubjectId VARCHAR(50);

DECLARE request_cursor CURSOR FOR
SELECT StudentId, SubjectId FROM SubjectRequest;

OPEN request_cursor;
FETCH NEXT FROM request_cursor INTO @StudentId, @RequestedSubjectId;

WHILE @@FETCH_STATUS = 0
BEGIN
    -- Check if student exists in SubjectAllotments
    IF EXISTS (
        SELECT 1 FROM SubjectAllotments WHERE StudentId = @StudentId
    )
    BEGIN
        -- Get the current active subject (is_valid = 1)
        SELECT @CurrentSubjectId = SubjectId
        FROM SubjectAllotments
        WHERE StudentId = @StudentId AND Is_Valid = 1;

        -- If requested subject is different from the current valid one
        IF @CurrentSubjectId IS NULL OR @CurrentSubjectId <> @RequestedSubjectId
        BEGIN
            -- Mark current active subject as invalid
            UPDATE SubjectAllotments
            SET Is_Valid = 0
            WHERE StudentId = @StudentId AND Is_Valid = 1;

            -- Insert the new requested subject as valid
            INSERT INTO SubjectAllotments (StudentId, SubjectId, Is_Valid)
            VALUES (@StudentId, @RequestedSubjectId, 1);
        END
        -- else do nothing (subject is already valid)
    END
    ELSE
    BEGIN
        -- Student does not exist, insert new record as valid
        INSERT INTO SubjectAllotments (StudentId, SubjectId, Is_Valid)
        VALUES (@StudentId, @RequestedSubjectId, 1);
    END

    FETCH NEXT FROM request_cursor INTO @StudentId, @RequestedSubjectId;
END

CLOSE request_cursor;
DEALLOCATE request_cursor;

-- Commit the transaction
COMMIT;
