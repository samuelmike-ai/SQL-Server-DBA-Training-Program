# Week 16: Database Development and Integration

## Learning Objectives
By the end of this week, you will be able to:
- Design and implement CI/CD pipelines for database development and deployment
- Establish version control strategies for database schema and code management
- Integrate SQL Server development with modern DevOps practices and tools
- Build automated testing frameworks for database changes
- Implement database development best practices for enterprise environments

## Introduction to Modern Database Development

The evolution of database development has accelerated dramatically with the rise of DevOps, continuous integration, and cloud-native applications. Modern enterprise organizations require database development practices that integrate seamlessly with application development workflows, support rapid iteration cycles, and maintain data integrity while enabling rapid deployment.

This week, we'll explore how database development has transformed from isolated, manual processes to integrated, automated workflows used by leading technology companies. We'll examine real-world scenarios from financial services firms deploying high-frequency trading system updates, to healthcare organizations managing patient data migrations, to retail companies coordinating inventory system deployments across thousands of stores.

Our focus will be on building robust, scalable development pipelines that ensure database changes are deployed safely, tested thoroughly, and rolled back quickly when necessary.

## Version Control for Database Development

### Database Schema Version Control Strategies

Modern database development requires sophisticated version control approaches that can handle complex schema changes while maintaining data integrity and supporting parallel development.

**Database-First vs Code-First vs Hybrid Approaches:**

Enterprise organizations often adopt hybrid approaches that combine the benefits of different strategies:

```sql
-- Create database version control management system
CREATE TABLE SchemaVersionControl (
    VersionID INT IDENTITY(1,1) PRIMARY KEY,
    VersionNumber NVARCHAR(50) NOT NULL, -- Semantic version: 1.2.3
    ChangeDescription NVARCHAR(500),
    ChangeType NVARCHAR(50), -- 'Feature', 'BugFix', 'Security', 'Performance'
    DatabaseName NVARCHAR(256),
    Environment NVARCHAR(50), -- 'Development', 'Staging', 'Production'
    DeploymentDate DATETIME2,
    DeployedBy NVARCHAR(256),
    RollbackScript NVARCHAR(MAX),
    TestCasesPassed INT DEFAULT 0,
    TestCasesTotal INT DEFAULT 0,
    RollbackRequired BIT DEFAULT 0,
    RollbackDate DATETIME2 NULL,
    RollbackReason NVARCHAR(500) NULL,
    PreDeploymentChecks NVARCHAR(MAX),
    PostDeploymentChecks NVARCHAR(MAX),
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    Status NVARCHAR(50) DEFAULT 'Pending', -- 'Pending', 'Deployed', 'Failed', 'RolledBack'
    CONSTRAINT UK_VersionControl UNIQUE (VersionNumber, DatabaseName, Environment)
);

-- Create table for tracking database objects and their versions
CREATE TABLE DatabaseObjectVersions (
    ObjectID BIGINT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(256),
    ObjectName NVARCHAR(256),
    ObjectType NVARCHAR(50), -- 'Table', 'View', 'StoredProcedure', 'Function', 'Trigger'
    SchemaName NVARCHAR(256),
    ObjectHash NVARCHAR(128), -- Hash of object definition for change detection
    VersionNumber NVARCHAR(50),
    Environment NVARCHAR(50),
    LastModified DATETIME2,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    IsSystemObject BIT DEFAULT 0,
    RequiresDataMigration BIT DEFAULT 0,
    CONSTRAINT UK_ObjectVersions UNIQUE (DatabaseName, ObjectName, SchemaName, Environment)
);

-- Create procedure for applying database migrations with rollback capability
CREATE PROCEDURE sp_ApplyDatabaseMigration
    @DatabaseName NVARCHAR(256),
    @VersionNumber NVARCHAR(50),
    @MigrationScript NVARCHAR(MAX),
    @Environment NVARCHAR(50) = 'Development',
    @EnableRollback BIT = 1,
    @PreMigrationChecks NVARCHAR(MAX) = NULL,
    @PostMigrationChecks NVARCHAR(MAX) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DeploymentStartTime DATETIME2 = GETDATE();
    DECLARE @DeploymentStatus NVARCHAR(50) = 'InProgress';
    DECLARE @ErrorMessage NVARCHAR(MAX) = '';
    DECLARE @ObjectsAffected INT = 0;
    DECLARE @ExecutionUser NVARCHAR(256) = SYSTEM_USER;
    DECLARE @CurrentVersion NVARCHAR(50);
    
    -- Validate environment and get current version
    SELECT TOP 1 @CurrentVersion = VersionNumber
    FROM SchemaVersionControl
    WHERE DatabaseName = @DatabaseName
      AND Environment = @Environment
      AND Status = 'Deployed'
    ORDER BY VersionNumber DESC;
    
    IF @CurrentVersion IS NULL
        SET @CurrentVersion = '0.0.0';
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- Validate version progression
        IF @VersionNumber <= @CurrentVersion
        BEGIN
            RAISERROR('Invalid version number. New version %s must be greater than current version %s', 16, 1, @VersionNumber, @CurrentVersion);
        END
        
        -- Execute pre-migration checks if provided
        IF @PreMigrationChecks IS NOT NULL
        BEGIN
            EXEC sp_executesql @PreMigrationChecks;
        END
        
        -- Create rollback backup script
        DECLARE @RollbackScript NVARCHAR(MAX);
        
        -- For table changes, create rollback script
        IF @MigrationScript LIKE '%CREATE TABLE%' OR @MigrationScript LIKE '%ALTER TABLE%'
        BEGIN
            -- Extract table names and create rollback statements
            SET @RollbackScript = dbo.fn_GenerateRollbackScript(@MigrationScript);
        END
        
        -- Execute migration script
        EXEC sp_executesql @MigrationScript;
        
        SET @ObjectsAffected = @@ROWCOUNT;
        
        -- Execute post-migration checks if provided
        IF @PostMigrationChecks IS NOT NULL
        BEGIN
            EXEC sp_executesql @PostMigrationChecks;
        END
        
        -- Update object versions tracking
        EXEC sp_UpdateObjectVersions @DatabaseName, @VersionNumber, @Environment;
        
        -- Record successful deployment
        INSERT INTO SchemaVersionControl (
            VersionNumber, ChangeDescription, DatabaseName, Environment,
            DeploymentDate, DeployedBy, RollbackScript, Status
        ) VALUES (
            @VersionNumber,
            CASE WHEN @MigrationScript LIKE '%CREATE%' THEN 'Object Creation'
                 WHEN @MigrationScript LIKE '%ALTER%' THEN 'Object Modification'
                 WHEN @MigrationScript LIKE '%DROP%' THEN 'Object Deletion'
                 ELSE 'Schema Change'
            END,
            @DatabaseName,
            @Environment,
            @DeploymentStartTime,
            @ExecutionUser,
            CASE WHEN @EnableRollback = 1 THEN @RollbackScript ELSE NULL END,
            'Deployed'
        );
        
        COMMIT TRANSACTION;
        SET @DeploymentStatus = 'Success';
        
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        SET @DeploymentStatus = 'Failed';
        SET @ErrorMessage = ERROR_MESSAGE();
        
        -- Log failed deployment
        INSERT INTO SchemaVersionControl (
            VersionNumber, ChangeDescription, DatabaseName, Environment,
            DeploymentDate, DeployedBy, Status, ErrorMessage
        ) VALUES (
            @VersionNumber,
            'Migration Failed',
            @DatabaseName,
            @Environment,
            @DeploymentStartTime,
            @ExecutionUser,
            'Failed',
            @ErrorMessage
        );
        
        -- Send notification for deployment failure
        EXEC sp_SendDeploymentNotification
            @DatabaseName = @DatabaseName,
            @VersionNumber = @VersionNumber,
            @Status = 'Failed',
            @ErrorMessage = @ErrorMessage;
    END CATCH
    
    -- Return deployment result
    SELECT 
        @DeploymentStatus as DeploymentStatus,
        @ObjectsAffected as ObjectsAffected,
        CONVERT(VARCHAR(30), @DeploymentStartTime, 120) as DeploymentTime,
        DATEDIFF(SECOND, @DeploymentStartTime, GETDATE()) as DurationSeconds,
        @ErrorMessage as ErrorMessage;
END;

-- Create function to generate rollback scripts
CREATE FUNCTION fn_GenerateRollbackScript
    @MigrationScript NVARCHAR(MAX)
RETURNS NVARCHAR(MAX)
AS
BEGIN
    DECLARE @RollbackScript NVARCHAR(MAX) = '';
    DECLARE @ObjectName NVARCHAR(256);
    DECLARE @SchemaName NVARCHAR(256);
    
    -- Parse migration script for table operations
    IF @MigrationScript LIKE '%CREATE TABLE%'
    BEGIN
        -- Extract table name and generate DROP TABLE
        SELECT @ObjectName = CASE 
            WHEN CHARINDEX('CREATE TABLE [', @MigrationScript) > 0 
            THEN SUBSTRING(@MigrationScript, 
                          CHARINDEX('CREATE TABLE [', @MigrationScript) + 14,
                          CHARINDEX(']', @MigrationScript, CHARINDEX('CREATE TABLE [', @MigrationScript) + 14) - CHARINDEX('CREATE TABLE [', @MigrationScript) - 14)
            ELSE NULL
        END;
        
        IF @ObjectName IS NOT NULL
            SET @RollbackScript = CONCAT('DROP TABLE [', @ObjectName, '];');
    END
    
    IF @MigrationScript LIKE '%ALTER TABLE%'
    BEGIN
        -- Generate script to drop new columns or revert constraints
        -- This is simplified - real implementations would parse the ALTER statement
        SET @RollbackScript = '-- Rollback script for ALTER TABLE would be generated here';
    END
    
    RETURN @RollbackScript;
END;
```

### Source Control Integration Patterns

Modern database development integrates tightly with Git and other version control systems. The following pattern shows how to manage database scripts in source control:

```bash
# Database Development Directory Structure
/database-project/
├── migrations/
│   ├── 001_create_customers_table.sql
│   ├── 002_add_customer_indexes.sql
│   ├── 003_add_customer_preferences.sql
│   └── rollback/
│       ├── 003_rollback_customer_preferences.sql
│       ├── 002_rollback_customer_indexes.sql
│       └── 001_rollback_customers_table.sql
├── stored-procedures/
│   ├── get-customer-data.sql
│   └── update-customer-profile.sql
├── views/
│   └── customer-summary-view.sql
├── functions/
│   └── calculate-customer-score.sql
├── triggers/
│   └── audit-customer-changes.sql
├── data/
│   └── seed-reference-data.sql
├── tests/
│   ├── unit-tests/
│   │   ├── test-customer-procedures.sql
│   │   └── test-customer-functions.sql
│   ├── integration-tests/
│   │   ├── test-customer-workflow.sql
│   │   └── test-customer-reporting.sql
│   └── performance-tests/
│       ├── test-customer-queries-performance.sql
│       └── test-customer-batch-operations.sql
├── scripts/
│   ├── deployment-preparation.sql
│   ├── database-maintenance.sql
│   └── data-migration-helpers.sql
└── documentation/
    ├── database-schema-documentation.md
    ├── api-documentation.md
    └── deployment-guide.md
```

### Git-Based Database Development Workflow

```bash
#!/bin/bash
# database-deployment-workflow.sh - Automated database deployment script

# Function to validate migration script
validate_migration() {
    local migration_file="$1"
    echo "Validating migration script: $migration_file"
    
    # Check for SQL injection patterns
    if grep -E "(EXEC\(|EXECUTE\(|sp_executesql)" "$migration_file" > /dev/null; then
        echo "WARNING: Dynamic SQL detected in $migration_file"
    fi
    
    # Check for DROP statements in production migrations
    if [[ "$BRANCH_NAME" == "main" ]] && grep -i "DROP " "$migration_file" > /dev/null; then
        echo "ERROR: DROP statements not allowed in production migrations"
        exit 1
    fi
    
    # Validate SQL syntax using sqlcmd (requires sqlcmd to be installed)
    if command -v sqlcmd &> /dev/null; then
        sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "SET PARSEONLY ON; $(cat $migration_file)" -b
    fi
}

# Function to apply migration
apply_migration() {
    local migration_file="$1"
    local version_number=$(basename "$migration_file" | cut -d'_' -f1)
    
    echo "Applying migration $version_number..."
    
    # Execute migration
    if command -v sqlcmd &> /dev/null; then
        sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -i "$migration_file" -b
        if [ $? -eq 0 ]; then
            echo "Migration $version_number applied successfully"
        else
            echo "ERROR: Migration $version_number failed"
            exit 1
        fi
    fi
}

# Function to run tests
run_tests() {
    local test_directory="$1"
    
    echo "Running database tests from $test_directory..."
    
    # Run unit tests
    for test_file in "$test_directory/unit-tests"/*.sql; do
        if [ -f "$test_file" ]; then
            echo "Running test: $(basename "$test_file")"
            # Execute test and capture results
            if command -v sqlcmd &> /dev/null; then
                sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -i "$test_file" -b
            fi
        fi
    done
}

# Main deployment workflow
main() {
    local migration_file="$1"
    local environment="${2:-development}"
    
    echo "Starting database deployment for $environment environment"
    
    # Validate environment variables
    if [ -z "$SQL_SERVER" ] || [ -z "$DATABASE" ]; then
        echo "ERROR: SQL_SERVER and DATABASE environment variables must be set"
        exit 1
    fi
    
    # Pre-deployment checks
    echo "Running pre-deployment checks..."
    
    # Validate migration script
    validate_migration "$migration_file"
    
    # Run unit tests
    run_tests "./tests"
    
    # Create backup before deployment
    echo "Creating database backup..."
    backup_timestamp=$(date +"%Y%m%d_%H%M%S")
    sqlcmd -S "$SQL_SERVER" -Q "BACKUP DATABASE [$DATABASE] TO DISK = 'C:\Backups\${DATABASE}_${backup_timestamp}.bak'"
    
    # Apply migration
    apply_migration "$migration_file"
    
    # Run post-deployment tests
    echo "Running post-deployment validation..."
    run_tests "./tests/integration"
    
    # Update version tracking
    echo "Updating version tracking..."
    sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "
        INSERT INTO SchemaVersionControl (VersionNumber, Environment, DeploymentDate, Status)
        VALUES ('$(basename "$migration_file" | cut -d'_' -f1)', '$environment', GETDATE(), 'Deployed')
    "
    
    echo "Database deployment completed successfully"
}

# Execute main function
main "$@"
```

## Continuous Integration for Database Development

### Automated Testing Frameworks

Enterprise database development requires comprehensive testing strategies that cover unit testing, integration testing, performance testing, and data validation.

**Database Unit Testing Framework:**

```sql
-- Create comprehensive unit testing framework
CREATE TABLE TestFramework (
    TestFrameworkID INT IDENTITY(1,1) PRIMARY KEY,
    TestSuiteName NVARCHAR(256),
    TestCaseName NVARCHAR(256),
    TestProcedure NVARCHAR(256), -- Name of stored procedure to test
    ExpectedResult NVARCHAR(MAX),
    TestDataSetup NVARCHAR(MAX),
    TestCleanup NVARCHAR(MAX),
    Environment NVARCHAR(50), -- 'Development', 'CI', 'Production'
    LastExecuted DATETIME2,
    ExecutionDuration INT,
    PassCount INT DEFAULT 0,
    FailCount INT DEFAULT 0,
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 DEFAULT GETDATE()
);

-- Create test execution tracking
CREATE TABLE TestExecutionLog (
    ExecutionID BIGINT IDENTITY(1,1) PRIMARY KEY,
    TestFrameworkID INT,
    TestSuiteName NVARCHAR(256),
    TestCaseName NVARCHAR(256),
    ExecutionStart DATETIME2,
    ExecutionEnd DATETIME2 NULL,
    Status NVARCHAR(50), -- 'Passed', 'Failed', 'Error', 'Skipped'
    ResultMessage NVARCHAR(MAX),
    ExpectedResult NVARCHAR(MAX),
    ActualResult NVARCHAR(MAX),
    Environment NVARCHAR(50),
    BuildNumber NVARCHAR(50),
    BranchName NVARCHAR(100),
    CommittedBy NVARCHAR(256),
    ExecutionDuration INT, -- in seconds
    IsPerformanceTest BIT DEFAULT 0,
    PerformanceBaseline DECIMAL(18,6) NULL,
    PerformanceResult DECIMAL(18,6) NULL,
    CONSTRAINT FK_TestExecution FOREIGN KEY (TestFrameworkID) REFERENCES TestFramework(TestFrameworkID)
);

-- Create procedure for executing unit tests
CREATE PROCEDURE sp_ExecuteDatabaseUnitTests
    @TestSuite NVARCHAR(256) = NULL,
    @Environment NVARCHAR(50) = 'CI',
    @BuildNumber NVARCHAR(50) = NULL,
    @BranchName NVARCHAR(100) = NULL
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ExecutionStartTime DATETIME2 = GETDATE();
    DECLARE @TotalTests INT = 0;
    DECLARE @PassedTests INT = 0;
    DECLARE @FailedTests INT = 0;
    DECLARE @ErrorTests INT = 0;
    DECLARE @ExecutionID BIGINT;
    
    -- Create temporary table for test results
    CREATE TABLE #TestResults (
        TestFrameworkID INT,
        TestSuiteName NVARCHAR(256),
        TestCaseName NVARCHAR(256),
        Status NVARCHAR(50),
        Message NVARCHAR(MAX),
        Duration INT
    );
    
    -- Execute tests
    DECLARE @CurrentTestID INT;
    DECLARE @CurrentTestSuite NVARCHAR(256);
    DECLARE @CurrentTestCase NVARCHAR(256);
    DECLARE @CurrentProcedure NVARCHAR(256);
    DECLARE @TestDataSetup NVARCHAR(MAX);
    DECLARE @TestCleanup NVARCHAR(MAX);
    DECLARE @ExecutionUser NVARCHAR(256) = SYSTEM_USER;
    
    DECLARE test_cursor CURSOR FOR
    SELECT 
        tf.TestFrameworkID,
        tf.TestSuiteName,
        tf.TestCaseName,
        tf.TestProcedure,
        tf.TestDataSetup,
        tf.TestCleanup
    FROM TestFramework tf
    WHERE tf.IsActive = 1
      AND (@TestSuite IS NULL OR tf.TestSuiteName = @TestSuite)
      AND tf.Environment = @Environment
    ORDER BY tf.TestSuiteName, tf.TestCaseName;
    
    OPEN test_cursor;
    FETCH NEXT FROM test_cursor INTO 
        @CurrentTestID, @CurrentTestSuite, @CurrentTestCase, @CurrentProcedure, @TestDataSetup, @TestCleanup;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        DECLARE @TestExecutionStart DATETIME2 = GETDATE();
        DECLARE @TestStatus NVARCHAR(50) = 'Failed';
        DECLARE @TestMessage NVARCHAR(MAX) = '';
        DECLARE @TestDuration INT;
        
        BEGIN TRY
            SET @TotalTests = @TotalTests + 1;
            
            -- Set up test data
            IF @TestDataSetup IS NOT NULL
                EXEC sp_executesql @TestDataSetup;
            
            -- Execute the test procedure and capture results
            DECLARE @TestResult NVARCHAR(MAX);
            
            IF @CurrentProcedure LIKE 'sp_%'
            BEGIN
                -- For stored procedures, execute and capture result
                DECLARE @ResultSQL NVARCHAR(MAX) = CONCAT('
                    DECLARE @TestResultValue NVARCHAR(MAX);
                    
                    -- Execute procedure and get result
                    EXEC @TestResultValue = ', @CurrentProcedure, ' @TestResult OUTPUT;
                    
                    SELECT @TestResultValue as TestResult;
                ');
                
                CREATE TABLE #TempResult (TestResult NVARCHAR(MAX));
                INSERT INTO #TempResult EXEC sp_executesql @ResultSQL;
                
                SELECT @TestResult = TestResult FROM #TempResult;
                DROP TABLE #TempResult;
            END
            ELSE
            BEGIN
                -- For direct queries, execute and capture result
                SET @TestResult = CAST((SELECT COUNT(*) FROM (EXEC @CurrentProcedure)) AS NVARCHAR(MAX));
            END
            
            -- Validate result (simplified - real implementation would have complex validation)
            IF @TestResult IS NOT NULL
            BEGIN
                SET @TestStatus = 'Passed';
                SET @PassedTests = @PassedTests + 1;
                SET @TestMessage = 'Test executed successfully';
            END
            ELSE
            BEGIN
                SET @TestStatus = 'Failed';
                SET @FailedTests = @FailedTests + 1;
                SET @TestMessage = 'Test returned unexpected result';
            END
            
            -- Clean up test data
            IF @TestCleanup IS NOT NULL
                EXEC sp_executesql @TestCleanup;
                
        END TRY
        BEGIN CATCH
            SET @TestStatus = 'Error';
            SET @ErrorTests = @ErrorTests + 1;
            SET @TestMessage = CONCAT('Error: ', ERROR_MESSAGE());
        END CATCH
        
        SET @TestDuration = DATEDIFF(SECOND, @TestExecutionStart, GETDATE());
        
        -- Log test execution
        INSERT INTO TestExecutionLog (
            TestFrameworkID, TestSuiteName, TestCaseName,
            ExecutionStart, ExecutionEnd, Status, ResultMessage,
            ExpectedResult, ActualResult, Environment, BuildNumber,
            BranchName, CommittedBy, ExecutionDuration
        ) VALUES (
            @CurrentTestID, @CurrentTestSuite, @CurrentTestCase,
            @TestExecutionStart, GETDATE(), @TestStatus, @TestMessage,
            'Expected result defined', @TestResult, @Environment, @BuildNumber,
            @BranchName, @ExecutionUser, @TestDuration
        );
        
        -- Store result in temporary table for summary
        INSERT INTO #TestResults VALUES (
            @CurrentTestID, @CurrentTestSuite, @CurrentTestCase,
            @TestStatus, @TestMessage, @TestDuration
        );
        
        FETCH NEXT FROM test_cursor INTO 
            @CurrentTestID, @CurrentTestSuite, @CurrentTestCase, @CurrentProcedure, @TestDataSetup, @TestCleanup;
    END;
    
    CLOSE test_cursor;
    DEALLOCATE test_cursor;
    
    DECLARE @ExecutionEndTime DATETIME2 = GETDATE();
    DECLARE @TotalDuration INT = DATEDIFF(SECOND, @ExecutionStartTime, @ExecutionEndTime);
    
    -- Return execution summary
    SELECT 
        @TotalTests as TotalTests,
        @PassedTests as PassedTests,
        @FailedTests as FailedTests,
        @ErrorTests as ErrorTests,
        CASE 
            WHEN @TotalTests > 0 
            THEN ROUND((@PassedTests * 100.0 / @TotalTests), 2)
            ELSE 0
        END as SuccessRate,
        @TotalDuration as TotalDurationSeconds,
        CASE 
            WHEN @FailedTests = 0 AND @ErrorTests = 0 THEN 'All Tests Passed'
            WHEN @PassedTests > 0 THEN 'Some Tests Failed'
            ELSE 'All Tests Failed'
        END as OverallStatus;
    
    -- Return detailed test results
    SELECT * FROM #TestResults ORDER BY TestSuiteName, TestCaseName;
    
    -- Clean up
    DROP TABLE #TestResults;
    
    -- Set exit code based on results
    IF @FailedTests > 0 OR @ErrorTests > 0
        RAISERROR('Database unit tests failed. Check test execution log for details.', 16, 1);
END;
```

### Integration Testing for Database Changes

Integration testing ensures that database changes work correctly with application code and other system components.

**Database Integration Testing Framework:**

```sql
-- Create integration test scenarios for database development
CREATE TABLE IntegrationTestScenarios (
    ScenarioID INT IDENTITY(1,1) PRIMARY KEY,
    ScenarioName NVARCHAR(256),
    TestType NVARCHAR(50), -- 'API_Integration', 'DataFlow', 'EndToEnd', 'Performance'
    DatabaseComponent NVARCHAR(256),
    ApplicationComponent NVARCHAR(256),
    TestDataSetup NVARCHAR(MAX),
    TestSteps NVARCHAR(MAX),
    ExpectedResults NVARCHAR(MAX),
    CleanupSteps NVARCHAR(MAX),
    Environment NVARCHAR(50),
    IsAutomated BIT DEFAULT 1,
    IsActive BIT DEFAULT 1,
    CreatedDate DATETIME2 DEFAULT GETDATE(),
    LastExecuted DATETIME2
);

-- Create integration test execution tracking
CREATE TABLE IntegrationTestExecution (
    ExecutionID BIGINT IDENTITY(1,1) PRIMARY KEY,
    ScenarioID INT,
    ScenarioName NVARCHAR(256),
    ExecutionStart DATETIME2,
    ExecutionEnd DATETIME2 NULL,
    Status NVARCHAR(50), -- 'Passed', 'Failed', 'Timeout', 'Error'
    BuildNumber NVARCHAR(50),
    TestEnvironment NVARCHAR(50),
    DatabaseVersion NVARCHAR(50),
    ApplicationVersion NVARCHAR(50),
    ErrorDetails NVARCHAR(MAX),
    PerformanceMetrics NVARCHAR(MAX),
    LogsLocation NVARCHAR(500),
    ExecutionDuration INT, -- in seconds
    CONSTRAINT FK_IntegrationTest FOREIGN KEY (ScenarioID) REFERENCES IntegrationTestScenarios(ScenarioID)
);

-- Create procedure for running integration tests
CREATE PROCEDURE sp_RunIntegrationTests
    @BuildNumber NVARCHAR(50),
    @TestEnvironment NVARCHAR(50) = 'CI',
    @DatabaseVersion NVARCHAR(50),
    @ApplicationVersion NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @ExecutionStartTime DATETIME2 = GETDATE();
    DECLARE @TotalScenarios INT = 0;
    DECLARE @PassedScenarios INT = 0;
    DECLARE @FailedScenarios INT = 0;
    DECLARE @CurrentScenario INT;
    
    -- Get active integration test scenarios
    CREATE TABLE #TestScenarios (
        ScenarioID INT,
        ScenarioName NVARCHAR(256),
        TestType NVARCHAR(50),
        DatabaseComponent NVARCHAR(256),
        ApplicationComponent NVARCHAR(256),
        TestDataSetup NVARCHAR(MAX),
        TestSteps NVARCHAR(MAX),
        ExpectedResults NVARCHAR(MAX),
        CleanupSteps NVARCHAR(MAX)
    );
    
    INSERT INTO #TestScenarios
    SELECT 
        ScenarioID, ScenarioName, TestType, DatabaseComponent, ApplicationComponent,
        TestDataSetup, TestSteps, ExpectedResults, CleanupSteps
    FROM IntegrationTestScenarios
    WHERE IsActive = 1
      AND IsAutomated = 1
      AND Environment = @TestEnvironment;
    
    -- Execute integration tests
    DECLARE @ScenarioID INT;
    DECLARE @ScenarioName NVARCHAR(256);
    DECLARE @TestType NVARCHAR(50);
    DECLARE @DatabaseComponent NVARCHAR(256);
    DECLARE @ApplicationComponent NVARCHAR(256);
    DECLARE @TestSteps NVARCHAR(MAX);
    DECLARE @ExpectedResults NVARCHAR(MAX);
    DECLARE @CleanupSteps NVARCHAR(MAX);
    
    DECLARE scenario_cursor CURSOR FOR
    SELECT 
        ScenarioID, ScenarioName, TestType, DatabaseComponent, ApplicationComponent,
        TestSteps, ExpectedResults, CleanupSteps
    FROM #TestScenarios;
    
    OPEN scenario_cursor;
    FETCH NEXT FROM scenario_cursor INTO 
        @ScenarioID, @ScenarioName, @TestType, @DatabaseComponent, @ApplicationComponent,
        @TestSteps, @ExpectedResults, @CleanupSteps;
    
    WHILE @@FETCH_STATUS = 0
    BEGIN
        DECLARE @ScenarioExecutionStart DATETIME2 = GETDATE();
        DECLARE @ScenarioStatus NVARCHAR(50) = 'Failed';
        DECLARE @ErrorDetails NVARCHAR(MAX) = '';
        DECLARE @PerformanceMetrics NVARCHAR(MAX) = '';
        DECLARE @ExecutionDuration INT;
        
        BEGIN TRY
            SET @TotalScenarios = @TotalScenarios + 1;
            
            -- Set up test data if required
            IF @TestDataSetup IS NOT NULL
                EXEC sp_executesql @TestDataSetup;
            
            -- Execute integration test steps based on test type
            IF @TestType = 'API_Integration'
            BEGIN
                -- For API integration tests, execute API calls and validate database state
                EXEC sp_ExecuteAPIIntegrationTest 
                    @DatabaseComponent = @DatabaseComponent,
                    @ApplicationComponent = @ApplicationComponent,
                    @TestSteps = @TestSteps;
            END
            ELSE IF @TestType = 'DataFlow'
            BEGIN
                -- For data flow tests, validate data movement and transformations
                EXEC sp_ExecuteDataFlowTest
                    @DatabaseComponent = @DatabaseComponent,
                    @TestSteps = @TestSteps;
            END
            ELSE IF @TestType = 'EndToEnd'
            BEGIN
                -- For end-to-end tests, execute complete workflows
                EXEC sp_ExecuteEndToEndTest
                    @TestSteps = @TestSteps,
                    @ExpectedResults = @ExpectedResults;
            END
            ELSE IF @TestType = 'Performance'
            BEGIN
                -- For performance tests, measure query execution times
                EXEC sp_ExecutePerformanceTest
                    @DatabaseComponent = @DatabaseComponent,
                    @PerformanceMetrics = @PerformanceMetrics OUTPUT;
            END
            
            SET @ScenarioStatus = 'Passed';
            SET @PassedScenarios = @PassedScenarios + 1;
            
        END TRY
        BEGIN CATCH
            SET @ScenarioStatus = 'Failed';
            SET @FailedScenarios = @FailedScenarios + 1;
            SET @ErrorDetails = ERROR_MESSAGE();
        END CATCH
        
        SET @ExecutionDuration = DATEDIFF(SECOND, @ScenarioExecutionStart, GETDATE());
        
        -- Log integration test execution
        INSERT INTO IntegrationTestExecution (
            ScenarioID, ScenarioName, ExecutionStart, ExecutionEnd, Status,
            BuildNumber, TestEnvironment, DatabaseVersion, ApplicationVersion,
            ErrorDetails, PerformanceMetrics, ExecutionDuration
        ) VALUES (
            @ScenarioID, @ScenarioName, @ScenarioExecutionStart, GETDATE(), @ScenarioStatus,
            @BuildNumber, @TestEnvironment, @DatabaseVersion, @ApplicationVersion,
            @ErrorDetails, @PerformanceMetrics, @ExecutionDuration
        );
        
        -- Clean up test data
        IF @CleanupSteps IS NOT NULL
            EXEC sp_executesql @CleanupSteps;
        
        FETCH NEXT FROM scenario_cursor INTO 
            @ScenarioID, @ScenarioName, @TestType, @DatabaseComponent, @ApplicationComponent,
            @TestSteps, @ExpectedResults, @CleanupSteps;
    END;
    
    CLOSE scenario_cursor;
    DEALLOCATE scenario_cursor;
    
    -- Update scenario last executed dates
    UPDATE IntegrationTestScenarios
    SET LastExecuted = GETDATE()
    WHERE ScenarioID IN (SELECT ScenarioID FROM #TestScenarios);
    
    -- Return integration test results
    SELECT 
        @TotalScenarios as TotalScenarios,
        @PassedScenarios as PassedScenarios,
        @FailedScenarios as FailedScenarios,
        CASE 
            WHEN @TotalScenarios > 0 
            THEN ROUND((@PassedScenarios * 100.0 / @TotalScenarios), 2)
            ELSE 0
        END as SuccessRate,
        CASE 
            WHEN @FailedScenarios = 0 THEN 'All Integration Tests Passed'
            WHEN @PassedScenarios > 0 THEN 'Some Integration Tests Failed'
            ELSE 'All Integration Tests Failed'
        END as OverallStatus;
    
    DROP TABLE #TestScenarios;
END;
```

## CI/CD Pipeline Implementation

### Automated Build and Deployment Pipeline

Modern database development requires sophisticated CI/CD pipelines that can automatically build, test, and deploy database changes.

**Jenkins Pipeline for Database Development:**

```groovy
// Jenkinsfile for database CI/CD pipeline
pipeline {
    agent any
    
    environment {
        SQL_SERVER = credentials('sql-server-address')
        SQL_USER = credentials('sql-user')
        SQL_PASSWORD = credentials('sql-password')
        DATABASE_DEV = credentials('database-dev-name')
        DATABASE_STAGING = credentials('database-staging-name')
        DATABASE_PROD = credentials('database-prod-name')
        BUILD_ARTIFACTS = 'database-artifacts'
        CREDENTIALS_ID = 'git-credentials'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out database code..."
                    checkout scm
                }
            }
        }
        
        stage('Code Analysis') {
            steps {
                script {
                    echo "Running static code analysis..."
                    
                    // Run SQL linting
                    sh '''
                        if command -v sqlfluff &> /dev/null; then
                            sqlfluff lint migrations/
                        fi
                    '''
                    
                    // Check for SQL injection patterns
                    sh '''
                        if grep -r "EXEC(" migrations/ > /dev/null; then
                            echo "WARNING: Dynamic SQL detected in migrations"
                        fi
                        
                        if grep -r "sp_executesql" migrations/ > /dev/null; then
                            echo "WARNING: Dynamic SQL detected in migrations"
                        fi
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    echo "Building database artifacts..."
                    
                    // Create build artifacts directory
                    sh "mkdir -p ${BUILD_ARTIFACTS}"
                    
                    // Copy migration scripts to artifacts
                    sh "cp -r migrations/ ${BUILD_ARTIFACTS}/"
                    sh "cp -r stored-procedures/ ${BUILD_ARTIFACTS}/"
                    sh "cp -r views/ ${BUILD_ARTIFACTS}/"
                    sh "cp -r functions/ ${BUILD_ARTIFACTS}/"
                    sh "cp -r triggers/ ${BUILD_ARTIFACTS}/"
                    sh "cp -r data/ ${BUILD_ARTIFACTS}/"
                    
                    // Create package metadata
                    sh '''
                        cat > ${BUILD_ARTIFACTS}/package.json << EOF
                        {
                            "name": "database-package",
                            "version": "${BUILD_NUMBER}",
                            "buildNumber": "${BUILD_NUMBER}",
                            "buildDate": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
                            "branch": "${BRANCH_NAME}",
                            "commit": "${GIT_COMMIT}"
                        }
                        EOF
                    '''
                    
                    // Archive artifacts
                    archiveArtifacts artifacts: BUILD_ARTIFACTS, fingerprint: true
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    echo "Running database unit tests..."
                    
                    // Test database connection
                    sh '''
                        if command -v sqlcmd &> /dev/null; then
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_DEV -Q "SELECT 1" -b
                        fi
                    '''
                    
                    // Run unit tests using stored procedure
                    sh '''
                        if command -v sqlcmd &> /dev/null; then
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_DEV \
                                   -Q "EXEC sp_ExecuteDatabaseUnitTests @Environment = 'CI', @BuildNumber = '${BUILD_NUMBER}', @BranchName = '${BRANCH_NAME}'"
                        fi
                    '''
                }
            }
        }
        
        stage('Integration Tests') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo "Running integration tests..."
                    
                    // Deploy to integration environment for testing
                    sh '''
                        echo "Deploying to integration environment..."
                        # Apply latest migration to integration database
                        for migration in migrations/*.sql; do
                            echo "Applying migration: $migration"
                            if command -v sqlcmd &> /dev/null; then
                                sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_DEV -i "$migration"
                            fi
                        done
                    '''
                    
                    // Run integration tests
                    sh '''
                        if command -v sqlcmd &> /dev/null; then
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_DEV \
                                   -Q "EXEC sp_RunIntegrationTests @BuildNumber = '${BUILD_NUMBER}', @DatabaseVersion = '${BUILD_NUMBER}'"
                        fi
                    '''
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo "Running security scans..."
                    
                    // Scan for sensitive data exposure
                    sh '''
                        echo "Scanning for potential security issues..."
                        
                        # Check for hardcoded passwords or connection strings
                        if grep -r -i -E "(password|secret|key).*=.*['\"][^'\"]+['\"]" migrations/ > /dev/null; then
                            echo "WARNING: Potential hardcoded credentials detected"
                        fi
                        
                        # Check for SELECT * queries
                        if grep -r "SELECT \\*" migrations/ > /dev/null; then
                            echo "WARNING: SELECT * queries detected - consider specifying columns"
                        fi
                        
                        # Check for implicit data type conversions
                        if grep -r -i -E "(varchar|char|nvarchar).*with.*implicit" migrations/ > /dev/null; then
                            echo "WARNING: Implicit data type conversions detected"
                        fi
                    '''
                }
            }
        }
        
        stage('Performance Tests') {
            when {
                anyOf {
                    branch 'release/*'
                    branch 'main'
                }
            }
            steps {
                script {
                    echo "Running performance tests..."
                    
                    // Test query performance
                    sh '''
                        echo "Executing performance tests..."
                        
                        if command -v sqlcmd &> /dev/null; then
                            # Run performance baseline tests
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_DEV \
                                   -i tests/performance/test-customer-queries-performance.sql
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                script {
                    echo "Deploying to staging environment..."
                    
                    // Deploy to staging with approvals
                    timeout(time: 30, unit: 'MINUTES') {
                        input message: 'Deploy to staging environment?', ok: 'Deploy'
                    }
                    
                    sh '''
                        echo "Applying migrations to staging database..."
                        
                        # Apply migrations to staging
                        for migration in migrations/*.sql; do
                            echo "Applying migration: $migration"
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_STAGING -i "$migration"
                        done
                    '''
                    
                    // Run smoke tests in staging
                    sh '''
                        echo "Running smoke tests in staging..."
                        
                        if command -v sqlcmd &> /dev/null; then
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_STAGING \
                                   -Q "EXEC sp_SmokeTest @Environment = 'Staging'"
                        fi
                    '''
                }
            }
        }
        
        stage('Production Deploy') {
            when {
                anyOf {
                    branch 'main'
                }
            }
            steps {
                script {
                    echo "Production deployment requires manual approval..."
                    
                    // Production deployment requires multiple approvals
                    timeout(time: 60, unit: 'MINUTES') {
                        input message: 'Deploy to production? This will affect live data.', ok: 'Deploy Now'
                    }
                    
                    // Production deployment with rollback capability
                    sh '''
                        echo "Creating production backup before deployment..."
                        sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD \
                               -Q "BACKUP DATABASE [$DATABASE_PROD] TO DISK = 'C:\\Backups\\${DATABASE_PROD}_${BUILD_NUMBER}.bak'"
                        
                        echo "Applying migrations to production database..."
                        for migration in migrations/*.sql; do
                            echo "Applying migration: $migration"
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_PROD -i "$migration"
                        done
                    '''
                    
                    // Post-deployment verification
                    sh '''
                        echo "Running post-deployment verification..."
                        
                        if command -v sqlcmd &> /dev/null; then
                            sqlcmd -S $SQL_SERVER -U $SQL_USER -P $SQL_PASSWORD -d $DATABASE_PROD \
                                   -Q "EXEC sp_PostDeploymentVerification @BuildNumber = '${BUILD_NUMBER}'"
                        fi
                    '''
                }
            }
        }
    }
    
    post {
        success {
            script {
                echo "Pipeline completed successfully!"
                
                // Send success notifications
                emailext (
                    subject: "Database CI/CD Pipeline Success - ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                    body: """
                        Database deployment pipeline completed successfully.
                        
                        Build: ${env.BUILD_NUMBER}
                        Branch: ${env.BRANCH_NAME}
                        Commit: ${env.GIT_COMMIT}
                        
                        Artifacts: ${env.BUILD_URL}artifact/database-artifacts/
                    """,
                    to: '${DEFAULT_RECIPIENTS}'
                )
            }
        }
        
        failure {
            script {
                echo "Pipeline failed!"
                
                // Send failure notifications
                emailext (
                    subject: "Database CI/CD Pipeline Failed - ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                    body: """
                        Database deployment pipeline failed.
                        
                        Build: ${env.BUILD_NUMBER}
                        Branch: ${env.BRANCH_NAME}
                        Commit: ${env.GIT_COMMIT}
                        
                        Please check the build logs for details.
                        Build URL: ${env.BUILD_URL}
                    """,
                    to: '${DEFAULT_RECIPIENTS}'
                )
            }
        }
        
        unstable {
            script {
                echo "Pipeline completed with warnings..."
                
                // Send warning notifications
                emailext (
                    subject: "Database CI/CD Pipeline Warnings - ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                    body: """
                        Database deployment pipeline completed with warnings.
                        
                        Build: ${env.BUILD_NUMBER}
                        Branch: ${env.BRANCH_NAME}
                        
                        Please review the build logs for details.
                        Build URL: ${env.BUILD_URL}
                    """,
                    to: '${DEFAULT_RECIPIENTS}'
                )
            }
        }
    }
}
```

### Container-Based Database Development

Modern development increasingly uses containerization for consistent environments across development, testing, and production.

**Docker Compose for Database Development:**

```yaml
# docker-compose.yml for database development environment
version: '3.8'

services:
  sql-server:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sql-server-dev
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - sql-server-data:/var/opt/mssql
      - ./init-scripts:/docker-entrypoint-initdb.d
      - ./backup:/var/opt/mssql/backup
    networks:
      - database-network
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "$$SA_PASSWORD" -Q "SELECT 1"
      interval: 30s
      timeout: 3s
      retries: 5
      start_period: 40s

  sql-agent:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: sql-agent-dev
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourStrong@Passw0rd
      - MSSQL_PID=Developer
    depends_on:
      - sql-server
    volumes:
      - ./sql-agent-scripts:/sql-agent-scripts
      - ./backup:/var/opt/mssql/backup
    networks:
      - database-network
    entrypoint: /bin/bash -c "
      /opt/mssql/bin/sqlservr & 
      sleep 30
      /opt/mssql-tools/bin/sqlcmd -S sql-server -U SA -P \"$$SA_PASSWORD\" -i /sql-agent-scripts/setup-agent.sql
      tail -f /dev/null
    "

  # Development tools
  azure-data-studio:
    image: mcr.microsoft.com/azure-data-studio:latest
    container_name: azure-data-studio
    ports:
      - "8001:8001"
    volumes:
      - ./notebooks:/tmp/notebooks
      - ./connections:/tmp/connections
    networks:
      - database-network

  # Database administration container
  adminer:
    image: adminer
    container_name: db-adminer
    ports:
      - "8080:8080"
    environment:
      - ADMINER_DEFAULT_SERVER=sql-server
    networks:
      - database-network

volumes:
  sql-server-data:
    driver: local

networks:
  database-network:
    driver: bridge
```

**Database Initialization Scripts:**

```sql
-- init-scripts/01-create-database.sql
IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = 'DevelopmentDB')
BEGIN
    CREATE DATABASE DevelopmentDB;
END
GO

USE DevelopmentDB;
GO

-- Create development schema
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = 'Development')
    EXEC('CREATE SCHEMA Development');
GO

-- Create test data schema
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = 'Test')
    EXEC('CREATE SCHEMA Test');
GO

-- Create audit schema
IF NOT EXISTS (SELECT * FROM sys.schemas WHERE name = 'Audit')
    EXEC('CREATE SCHEMA Audit');
GO

-- Create initial development users
IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = 'DevUser')
BEGIN
    CREATE ROLE DevUser;
    ALTER ROLE DevUser ADD MEMBER [DevelopmentSchema];
END
GO

-- Create version control table
CREATE TABLE SchemaVersionControl (
    VersionID INT IDENTITY(1,1) PRIMARY KEY,
    VersionNumber NVARCHAR(50) NOT NULL,
    ChangeDescription NVARCHAR(500),
    DeploymentDate DATETIME2 DEFAULT GETDATE(),
    DeployedBy NVARCHAR(256) DEFAULT SYSTEM_USER,
    AppliedScript NVARCHAR(500),
    Status NVARCHAR(50) DEFAULT 'Applied'
);
GO
```

## DevOps Integration for Database Development

### Infrastructure as Code for Databases

Database infrastructure management has evolved to follow Infrastructure as Code (IaC) principles, enabling consistent, repeatable database deployments.

**Terraform Configuration for SQL Server Infrastructure:**

```hcl
# main.tf - Azure SQL Server Infrastructure
terraform {
  required_version = ">= 1.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

# Configure Azure Provider
provider "azurerm" {
  features {
    key_vault {
      purge_soft_delete_on_destroy = true
    }
  }
}

# Variables
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "database_admin_password" {
  description = "SQL Server admin password"
  type        = string
  sensitive   = true
}

# Resource Group
resource "azurerm_resource_group" "database_rg" {
  name     = "rg-sql-${var.environment}-${random_id.random_id.hex}"
  location = var.location
  tags = {
    Environment = var.environment
    Project     = "database-infrastructure"
    ManagedBy   = "terraform"
  }
}

# Key Vault for database credentials
resource "azurerm_key_vault" "database_kv" {
  name                        = "kv-sql-${var.environment}-${random_id.random_id.hex}"
  location                    = var.location
  resource_group_name         = azurerm_resource_group.database_rg.name
  enabled_for_disk_encryption = true
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  soft_delete_retention_days  = 7
  purge_protection_enabled    = false

  sku_name = "standard"

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = data.azurerm_client_config.current.object_id

    key_permissions = [
      "get",
      "list",
      "create",
      "delete",
      "update",
      "import",
      "purge",
      "recover",
      "backup",
      "restore"
    ]

    secret_permissions = [
      "get",
      "list",
      "set",
      "delete",
      "backup",
      "restore",
      "recover",
      "purge"
    ]
  }
}

# Store SQL Server password in Key Vault
resource "azurerm_key_vault_secret" "sql_password" {
  name         = "sql-password"
  value        = var.database_admin_password
  key_vault_id = azurerm_key_vault.database_kv.id
}

# Azure SQL Server
resource "azurerm_mssql_server" "sql_server" {
  name                         = "sql-${var.environment}-${random_id.random_id.hex}"
  location                     = var.location
  resource_group_name          = azurerm_resource_group.database_rg.name
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.database_admin_password
  minimum_tls_version          = "1.2"

  azuread_administrator {
    login_username = "SQL Admins"
    object_id      = data.azuread_group.sql_admins.id
  }

  tags = {
    Environment = var.environment
    Project     = "database-infrastructure"
  }
}

# Azure SQL Database
resource "azurerm_mssql_database" "application_db" {
  name           = "ApplicationDB"
  server_id      = azurerm_mssql_server.sql_server.id
  sku_name       = "GP_S_Gen5_2"
  zone_redundant = false
  max_size_gb    = 100
  sample_name    = null

  extended_auditing_policy {
    storage_endpoint                        = azurerm_storage_account.database_backup.primary_blob_endpoint
    storage_account_access_key              = azurerm_storage_account.database_backup.primary_access_key
    retention_in_days                       = 90
  }

  threat_detection_policy {
    enabled              = true
    storage_account_access_key = azurerm_storage_account.database_backup.primary_access_key
    storage_endpoint      = azurerm_storage_account.database_backup.primary_blob_endpoint
    retention_days       = 90
    
    alert_threshold = 15
    disabled_alerts = ["Sql_Injection", "Data_Exfiltration"]
  }

  tags = {
    Environment = var.environment
    Database    = "Production"
  }
}

# Backup Storage Account
resource "azurerm_storage_account" "database_backup" {
  name                     = "sqldbbak${random_id.random_id.hex}"
  location                 = var.location
  resource_group_name      = azurerm_resource_group.database_rg.name
  account_tier             = "Standard"
  account_replication_type = "GRS"
  min_tls_version          = "TLS1_2"

  blob_properties {
    versioning_enabled = true
    delete_retention_policy {
      days = 90
    }
  }

  network_rules {
    default_action = "Deny"
    bypass         = ["AzureServices"]
    
    ip_rules = [
      var.allowed_ip_cidr
    ]
  }

  tags = {
    Environment = var.environment
    Backup      = "Database"
  }
}

# Network Security Group for SQL Server
resource "azurerm_network_security_group" "sql_nsg" {
  name                = "nsg-sql-${var.environment}"
  location            = var.location
  resource_group_name = azurerm_resource_group.database_rg.name

  security_rule {
    name                       = "SQL"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "1433"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = {
    Environment = var.environment
    Service     = "SQL Server"
  }
}

# Generate random ID for unique resource names
resource "random_id" "random_id" {
  byte_length = 4
}

# Data sources
data "azurerm_client_config" "current" {}

data "azuread_group" "sql_admins" {
  display_name = "SQL Administrators"
}
```

### GitOps for Database Development

GitOps principles apply to database development, ensuring that all database changes are version-controlled, reviewable, and deployable through automated processes.

**GitOps Workflow Implementation:**

```bash
#!/bin/bash
# database-gitops-workflow.sh - GitOps workflow for database changes

set -e

# Configuration
ENVIRONMENT="${1:-dev}"
BRANCH_NAME="${2:-$(git branch --show-current)}"
MIGRATION_FILES_PATH="migrations"
BACKUP_RETENTION_DAYS=7

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Function to validate migration script
validate_migration() {
    local migration_file="$1"
    
    log_info "Validating migration: $migration_file"
    
    # Check for dangerous operations in production
    if [[ "$ENVIRONMENT" == "prod" ]]; then
        if grep -iE "(DROP (TABLE|DATABASE)|ALTER.*DROP|TRUNCATE TABLE)" "$migration_file"; then
            log_error "Dangerous operation detected in production migration: $migration_file"
            return 1
        fi
    fi
    
    # Check for SQL injection patterns
    if grep -E "(EXEC\(|EXECUTE\(|sp_executesql)" "$migration_file" > /dev/null; then
        log_warn "Dynamic SQL detected in migration: $migration_file"
    fi
    
    # Validate SQL syntax (if sqlcmd is available)
    if command -v sqlcmd &> /dev/null && [[ "$ENVIRONMENT" != "prod" ]]; then
        sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "SET PARSEONLY ON; $(cat $migration_file)" -b > /dev/null
        if [[ $? -eq 0 ]]; then
            log_info "SQL syntax validation passed for: $migration_file"
        else
            log_error "SQL syntax validation failed for: $migration_file"
            return 1
        fi
    fi
    
    return 0
}

# Function to apply migration
apply_migration() {
    local migration_file="$1"
    local version=$(basename "$migration_file" | cut -d'_' -f1)
    
    log_info "Applying migration version: $version"
    
    # Create rollback script
    local rollback_file="rollback/${version}_rollback.sql"
    if [[ ! -f "$rollback_file" ]]; then
        rollback_file=""
        log_warn "No rollback script found for version: $version"
    fi
    
    # Apply migration
    if command -v sqlcmd &> /dev/null; then
        sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -i "$migration_file" -b
        if [[ $? -eq 0 ]]; then
            log_info "Migration $version applied successfully"
            
            # Record deployment
            sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "
                INSERT INTO SchemaVersionControl (VersionNumber, Environment, DeploymentDate, Status, RollbackScript)
                VALUES ('$version', '$ENVIRONMENT', GETDATE(), 'Deployed', '$rollback_file')
            "
            return 0
        else
            log_error "Migration $version failed to apply"
            return 1
        fi
    else
        log_error "sqlcmd not available. Cannot apply migration."
        return 1
    fi
}

# Function to run tests
run_tests() {
    local test_type="$1"
    log_info "Running $test_type tests..."
    
    case "$test_type" in
        "unit")
            if command -v sqlcmd &> /dev/null; then
                sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "EXEC sp_ExecuteDatabaseUnitTests @Environment = '$ENVIRONMENT'"
            fi
            ;;
        "integration")
            if command -v sqlcmd &> /dev/null; then
                sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "EXEC sp_RunIntegrationTests @TestEnvironment = '$ENVIRONMENT'"
            fi
            ;;
        "smoke")
            if command -v sqlcmd &> /dev/null; then
                sqlcmd -S "$SQL_SERVER" -d "$DATABASE" -Q "EXEC sp_SmokeTest @Environment = '$ENVIRONMENT'"
            fi
            ;;
        *)
            log_error "Unknown test type: $test_type"
            return 1
            ;;
    esac
}

# Function to create backup
create_backup() {
    local backup_name="${DATABASE}_${ENVIRONMENT}_$(date +%Y%m%d_%H%M%S)"
    log_info "Creating backup: $backup_name"
    
    if command -v sqlcmd &> /dev/null; then
        sqlcmd -S "$SQL_SERVER" -Q "BACKUP DATABASE [$DATABASE] TO DISK = 'C:\Backups\$backup_name.bak'"
        if [[ $? -eq 0 ]]; then
            log_info "Backup created successfully: $backup_name"
            return 0
        else
            log_error "Backup creation failed"
            return 1
        fi
    fi
}

# Function to cleanup old backups
cleanup_backups() {
    log_info "Cleaning up backups older than $BACKUP_RETENTION_DAYS days"
    
    if command -v sqlcmd &> /dev/null; then
        sqlcmd -S "$SQL_SERVER" -Q "
            EXEC sp_delete_backup_history
                @database_name = '$DATABASE'
                ,@oldest_date = DATEADD(DAY, -$BACKUP_RETENTION_DAYS, GETDATE())
        "
    fi
}

# Main workflow
main() {
    log_info "Starting GitOps database deployment workflow"
    log_info "Environment: $ENVIRONMENT"
    log_info "Branch: $BRANCH_NAME"
    
    # Check required environment variables
    if [[ -z "$SQL_SERVER" ]] || [[ -z "$DATABASE" ]]; then
        log_error "SQL_SERVER and DATABASE environment variables must be set"
        exit 1
    fi
    
    # Validate branch protection
    if [[ "$ENVIRONMENT" == "prod" ]] && [[ "$BRANCH_NAME" != "main" ]]; then
        log_error "Production deployments must be from main branch"
        exit 1
    fi
    
    # Run pre-deployment validation
    log_info "Running pre-deployment validation..."
    
    for migration in "$MIGRATION_FILES_PATH"/*.sql; do
        if [[ -f "$migration" ]]; then
            if ! validate_migration "$migration"; then
                log_error "Migration validation failed: $migration"
                exit 1
            fi
        fi
    done
    
    # Run tests based on environment
    if [[ "$ENVIRONMENT" != "dev" ]]; then
        run_tests "unit"
        run_tests "integration"
    fi
    
    # Create backup for non-dev environments
    if [[ "$ENVIRONMENT" != "dev" ]]; then
        if ! create_backup; then
            log_error "Backup creation failed"
            exit 1
        fi
    fi
    
    # Apply migrations
    log_info "Applying migrations..."
    for migration in "$MIGRATION_FILES_PATH"/*.sql; do
        if [[ -f "$migration" ]]; then
            if ! apply_migration "$migration"; then
                log_error "Migration application failed"
                exit 1
            fi
        fi
    done
    
    # Run post-deployment tests
    run_tests "smoke"
    
    # Cleanup old backups
    cleanup_backups
    
    log_info "GitOps workflow completed successfully"
}

# Execute main function
main "$@"
```

## Database Development Best Practices

### Code Review and Quality Gates

Modern database development incorporates automated code review and quality gates similar to application development practices.

**SQL Code Quality Framework:**

```sql
-- Create code quality tracking system
CREATE TABLE CodeQualityMetrics (
    QualityID BIGINT IDENTITY(1,1) PRIMARY KEY,
    DatabaseName NVARCHAR(256),
    ObjectName NVARCHAR(256),
    ObjectType NVARCHAR(50),
    QualityScore DECIMAL(5,2), -- 0-100 score
    ComplexityScore INT, -- Cyclomatic complexity
    MaintainabilityScore DECIMAL(5,2), -- 0-100 maintainability score
    PerformanceScore DECIMAL(5,2), -- 0-100 performance score
    SecurityScore DECIMAL(5,2), -- 0-100 security score
    StyleViolations INT, -- Number of style violations
    TechnicalDebt INT, -- Hours of technical debt
    LastReviewed DATETIME2,
    ReviewStatus NVARCHAR(50), -- 'Pending', 'Approved', 'Rejected', 'RequiresChanges'
    Reviewer NVARCHAR(256),
    Comments NVARCHAR(MAX)
);

-- Create procedure for code quality analysis
CREATE PROCEDURE sp_AnalyzeCodeQuality
    @DatabaseName NVARCHAR(256),
    @ObjectName NVARCHAR(256) = NULL,
    @QualityThreshold DECIMAL(5,2) = 70.0
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Analyze stored procedures
    DECLARE @SQL NVARCHAR(MAX);
    
    SET @SQL = CONCAT('
        USE [', @DatabaseName, '];
        
        WITH ProcedureAnalysis AS (
            SELECT 
                p.name as ObjectName,
                ''StoredProcedure'' as ObjectType,
                LEN(p.definition) as CodeLength,
                LEN(p.definition) - LEN(REPLACE(p.definition, CHAR(10), '''')) as LineCount,
                CASE 
                    WHEN p.definition LIKE ''%SELECT *%'' THEN 1
                    ELSE 0
                END as HasSelectStar,
                CASE 
                    WHEN p.definition LIKE ''%NOLOCK%'' OR p.definition LIKE ''%WITH (NOLOCK)%'' THEN 1
                    ELSE 0
                END as HasNolock,
                CASE 
                    WHEN p.definition LIKE ''%EXEC(%'') OR p.definition LIKE ''%EXECUTE(%'') THEN 1
                    ELSE 0
                END as HasDynamicSQL,
                CASE 
                    WHEN p.definition LIKE ''%GO%'' THEN 1
                    ELSE 0
                END as HasGoStatements,
                CASE 
                    WHEN p.definition LIKE ''%sp_%'' THEN 1
                    ELSE 0
                END as UsesSystemProcedures
            FROM sys.procedures p
            WHERE (@ObjectName IS NULL OR p.name = @ObjectName)
        )
        SELECT 
            ObjectName,
            ObjectType,
            CodeLength,
            LineCount,
            HasSelectStar,
            HasNolock,
            HasDynamicSQL,
            HasGoStatements,
            UsesSystemProcedures,
            -- Calculate quality score
            100 - (
                HasSelectStar * 20 +
                HasNolock * 15 +
                HasDynamicSQL * 25 +
                HasGoStatements * 10 +
                UsesSystemProcedures * 15 +
                CASE WHEN LineCount > 500 THEN 20 ELSE LineCount * 0.04 END
            ) as QualityScore
        FROM ProcedureAnalysis
        ORDER BY QualityScore ASC;
    ');
    
    CREATE TABLE #QualityResults (
        ObjectName NVARCHAR(256),
        ObjectType NVARCHAR(50),
        CodeLength INT,
        LineCount INT,
        HasSelectStar BIT,
        HasNolock BIT,
        HasDynamicSQL BIT,
        HasGoStatements BIT,
        UsesSystemProcedures BIT,
        QualityScore DECIMAL(5,2)
    );
    
    INSERT INTO #QualityResults
    EXEC sp_executesql @SQL, N'@ObjectName NVARCHAR(256)', @ObjectName;
    
    -- Log quality analysis results
    INSERT INTO CodeQualityMetrics (
        DatabaseName, ObjectName, ObjectType, QualityScore,
        SecurityScore, ReviewStatus, LastReviewed
    )
    SELECT 
        @DatabaseName,
        qr.ObjectName,
        qr.ObjectType,
        qr.QualityScore,
        100 - (qr.HasSelectStar * 30 + qr.HasDynamicSQL * 40 + qr.HasNolock * 20) as SecurityScore,
        CASE 
            WHEN qr.QualityScore < @QualityThreshold THEN 'RequiresChanges'
            ELSE 'Pending'
        END as ReviewStatus,
        GETDATE() as LastReviewed
    FROM #QualityResults qr
    WHERE qr.QualityScore < @QualityThreshold;
    
    -- Return low quality objects for review
    SELECT TOP 50
        qr.ObjectName,
        qr.ObjectType,
        qr.QualityScore,
        qr.SecurityScore,
        qr.HasSelectStar,
        qr.HasDynamicSQL,
        qr.HasNolock,
        qr.LineCount,
        CASE 
            WHEN qr.HasSelectStar = 1 THEN 'Avoid SELECT *'
            WHEN qr.HasDynamicSQL = 1 THEN 'Review dynamic SQL usage'
            WHEN qr.HasNolock = 1 THEN 'Consider isolation level instead of NOLOCK'
            WHEN qr.LineCount > 500 THEN 'Procedure is too long - consider breaking it up'
            ELSE 'Code review required'
        END as Recommendation
    FROM #QualityResults qr
    WHERE qr.QualityScore < @QualityThreshold
    ORDER BY qr.QualityScore ASC;
    
    DROP TABLE #QualityResults;
END;
```

## Lab Exercises and Hands-On Scenarios

### Exercise 1: CI/CD Pipeline Implementation
Build a complete CI/CD pipeline for database development including:
- Git repository setup with database scripts and versioning
- Automated code analysis and security scanning
- Unit testing framework with stored procedure validation
- Integration testing with application components
- Deployment automation with rollback capabilities
- Quality gates and approval workflows

### Exercise 2: DevOps Integration Project
Create a DevOps-integrated database development workflow featuring:
- Infrastructure as Code implementation using Terraform
- Containerized development environment with Docker
- GitOps workflow for database change management
- Automated testing and deployment processes
- Monitoring and alerting integration
- Performance baseline establishment and regression testing

### Exercise 3: Enterprise Code Quality Framework
Implement a comprehensive code quality framework including:
- Automated code analysis and quality scoring
- Technical debt tracking and measurement
- Security vulnerability scanning
- Performance impact assessment
- Review workflow automation
- Quality trend analysis and reporting

## Week 16 Summary

This comprehensive week on database development and integration has covered the transformation of database development from isolated, manual processes to integrated, automated workflows. We've explored version control strategies, CI/CD pipelines, DevOps integration, and quality frameworks that enable organizations to deliver database changes safely and rapidly.

The key insight is that modern database development must embrace the same DevOps principles and practices that have revolutionized application development. Successful database development teams integrate tightly with development workflows, leverage automation for repetitive tasks, and maintain high standards for code quality and testing.

## Next Week Preview

Next week, we'll explore Cloud SQL Server deployments, covering Azure SQL Database, cloud migration strategies, hybrid cloud scenarios, and the unique considerations for managing SQL Server in cloud environments while maintaining enterprise-level performance, security, and availability.