# Gennovas.MaintenanceTask

# Maintenance Tasks SSIS Deployment Guide

This document describes the steps required to deploy and schedule the **Maintenance Tasks SSIS Project**.

---

## 1. Create Integration Services Catalogs

Use **SQL Server Management Studio (SSMS)** to create the SSIS catalog if it does not exist:

1. Open SSMS and connect to the target SQL Server instance.
2. Expand **Integration Services Catalogs**.
3. Right-click and select **Create Catalog...**.
4. Follow the wizard to enable **CLR Integration** and set a password for encryption.

> Note: The password is used to encrypt sensitive data in SSISDB. It does not need to be stored permanently. If an issue occurs, you can recreate the catalog.

---

## 2. Create Folder

1. In SSMS, navigate to **Integration Services Catalogs → SSISDB**.
2. Right-click **Folders** and select **Create Folder**.
3. Name the folder: `Maintenance`.

---

## 3. Deploy Project

Use **Microsoft Visual Studio** to deploy the SSIS project:

1. Open the project: `Gennovas.MaintenanceTask`.
2. Right-click the project → **Deploy...**
3. Choose the target server and the `Maintenance` folder.
4. Complete the wizard to deploy the `.ispac` file.

---

## 4. Create Environments

Use SSMS or T-SQL to create the environment `PRD` and environment variables.

```sql
USE SSISDB;
GO

-- Declare Folder and Environment
DECLARE 
    @FolderName NVARCHAR(128)        = N'Maintenance',
    @EnvironmentName NVARCHAR(128)   = N'PRD',
    @EnvironmentDesc NVARCHAR(512)   = N'Environment for Maintenance Tasks';

-- Declare Environment Variables
DECLARE 
    @ApplicationName NVARCHAR(512)   = N'TTC - Maintenance Task',
    @BackupAppDbName NVARCHAR(512)   = N'TTC_LIVE_App',
    @BackupFormDbName NVARCHAR(512)  = N'TTC_LIVE_Forms',
    @BackupObjectDbName NVARCHAR(512)= N'TTC_LIVE_Objects',
    @BackupSSRSDbName NVARCHAR(512)  = N'ReportServer',
    @DataSource NVARCHAR(512)        = N'TTC-SQLDB',
    @DaysToKeep INT                  = 3,
    @InitialCatalog NVARCHAR(512)    = N'master',
    @MinMemoryPct FLOAT              = 0.25,
    @OSReservedMB FLOAT              = 4096,
    @ReleaseMemoryPct FLOAT          = 0.1;

-- 1) Create Environment
IF NOT EXISTS (
    SELECT 1 
    FROM catalog.environments e
    INNER JOIN catalog.folders f ON e.folder_id = f.folder_id
    WHERE f.name = @FolderName AND e.name = @EnvironmentName
)
BEGIN
    EXEC catalog.create_environment 
        @folder_name = @FolderName,
        @environment_name = @EnvironmentName,
        @environment_description = @EnvironmentDesc;
END

-- 2) Create Environment Variables
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'ApplicationName', @sensitive=0, @description=N'Application name', @data_type=N'String', @value=@ApplicationName;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'BackupAppDbName', @sensitive=0, @description=N'Backup Application DB Name', @data_type=N'String', @value=@BackupAppDbName;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'BackupFormDbName', @sensitive=0, @description=N'Backup Forms DB Name', @data_type=N'String', @value=@BackupFormDbName;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'BackupObjectDbName', @sensitive=0, @description=N'Backup Objects DB Name', @data_type=N'String', @value=@BackupObjectDbName;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'BackupSSRSDbName', @sensitive=0, @description=N'Backup SSRS DB Name', @data_type=N'String', @value=@BackupSSRSDbName;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'DataSource', @sensitive=0, @description=N'Data Source', @data_type=N'String', @value=@DataSource;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'DaysToKeep', @sensitive=0, @description=N'Days to keep backups', @data_type=N'Int32', @value=@DaysToKeep;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'InitialCatalog', @sensitive=0, @description=N'Initial Catalog', @data_type=N'String', @value=@InitialCatalog;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'MinMemoryPct', @sensitive=0, @description=N'Minimum memory percent', @data_type=N'Double', @value=@MinMemoryPct;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'OSReservedMB', @sensitive=0, @description=N'OS Reserved Memory (MB)', @data_type=N'Double', @value=@OSReservedMB;
EXEC catalog.create_environment_variable @folder_name=@FolderName, @environment_name=@EnvironmentName, @variable_name=N'ReleaseMemoryPct', @sensitive=0, @description=N'Release memory percent', @data_type=N'Double', @value=@ReleaseMemoryPct;

-- 3) Reference Environment to Project (Optional)
DECLARE @ReferenceId BIGINT;
EXEC catalog.create_environment_reference
    @environment_name = @EnvironmentName,
    @reference_id     = @ReferenceId OUTPUT,
    @project_name     = N'Gennovas.MaintenanceTask',
    @reference_type   = N'R',  -- R = Required, A = Optional
    @folder_name      = @FolderName;
```
---

## 5. Map Environment Variables to Project Parameters

After creating the environment and deploying your SSIS project, you need to map environment variables to the project parameters. This ensures that the packages use the correct values at runtime.

### Steps

1. **Open SQL Server Management Studio (SSMS)**  
   Connect to the SQL Server instance where `SSISDB` is deployed.

2. **Navigate to the Project**  
   - Expand: `Integration Services Catalogs → SSISDB → [Your Folder] → Projects`  
   - Right-click the project (e.g., `Gennovas.MaintenanceTask`) and select **Configure**.

3. **Add Environment Reference**  
   - In the **Configure Project** window, go to the **References** page.  
   - Click **Add** to reference the environment (e.g., `PRD`) that you created.  
   - Choose the reference type:  
     - **Required (`R`)**: The project must use this environment.  
     - **Optional (`A`)**: The project can run without this environment.

4. **Map Project Parameters to Environment Variables**  
   - Go to the **Parameters** tab.  
   - For each project parameter that should be set from an environment variable:  
     - Click the **Value** column → **Use environment variable**.  
     - Select the corresponding environment variable (e.g., map `BackupAppDbName` parameter to `BackupAppDbName` environment variable).  
   - Repeat for all parameters that need dynamic values from the environment.

5. **Save Configuration**  
   - Click **OK** to save the environment reference and parameter mappings.  
   - The project is now configured to automatically use values from the environment during execution.

## Notes

- Environment variables allow you to maintain different configurations for **development, test, and production** environments without modifying the project itself.  
- Parameter mapping ensures consistency and reduces the risk of errors due to hard-coded values.  
- After mapping, any SQL Agent job or manual execution of the SSIS package will automatically pick up the environment values.

---

## 6. Create SQL Agent Job

Use SSMS or T-SQL to create a scheduled job for daily execution of maintenance packages:

```sql
USE [msdb]
GO

/****** Object:  Job [MaintenanceTasks - Every day]    Script Date: 22/08/2025 7:32:16 AM ******/
BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0
/****** Object:  JobCategory [[Uncategorized (Local)]]    Script Date: 22/08/2025 7:32:16 AM ******/
IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL', @name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

END

DECLARE @jobId BINARY(16),
	@serverName NVARCHAR(255) = N'TTC-SQLDB',
	@loginName NVARCHAR(255) = N'TTC-SQLDB\Administrator',
	@cmd NVARCHAR(3800) = N''

EXEC @ReturnCode =  msdb.dbo.sp_add_job @job_name=N'MaintenanceTasks - Every day', 
		@enabled=1, 
		@notify_level_eventlog=0, 
		@notify_level_email=0, 
		@notify_level_netsend=0, 
		@notify_level_page=0, 
		@delete_level=0, 
		@description=N'No description available.', 
		@category_name=N'[Uncategorized (Local)]', 
		@owner_login_name=@loginName, @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [Call MT01 - ShrinkDatabase]    Script Date: 22/08/2025 7:32:16 AM ******/
SET @cmd = 
    N'/ISSERVER "\"\SSISDB\Maintenance\Gennovas.MaintenanceTask\MT01 - ShrinkDatabase.dtsx\"" ' +
    N'/SERVER "\"' + @serverName + N'\"" ' +
    N'/ENVREFERENCE 1 ' +
    N'/Par "\"$ServerOption::LOGGING_LEVEL(Int16)\"";1 ' +
    N'/Par "\"$ServerOption::SYNCHRONIZED(Boolean)\"";True ' +
    N'/CALLERINFO SQLAGENT /REPORTING E';
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Call MT01 - ShrinkDatabase', 
		@step_id=1, 
		@cmdexec_success_code=0, 
		@on_success_action=3, 
		@on_success_step_id=0, 
		@on_fail_action=3, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'SSIS', 
		@command=@cmd,
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [Call MT02 - DatabaseFullBackup]    Script Date: 22/08/2025 7:32:17 AM ******/
SET @cmd = 
    N'/ISSERVER "\"\SSISDB\Maintenance\Gennovas.MaintenanceTask\MT02 - DatabaseFullBackup.dtsx\"" ' +
    N'/SERVER "\"' + @serverName + N'\"" ' +
    N'/ENVREFERENCE 1 ' +
    N'/Par "\"$ServerOption::LOGGING_LEVEL(Int16)\"";1 ' +
    N'/Par "\"$ServerOption::SYNCHRONIZED(Boolean)\"";True ' +
    N'/CALLERINFO SQLAGENT /REPORTING E';
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Call MT02 - DatabaseFullBackup', 
		@step_id=2, 
		@cmdexec_success_code=0, 
		@on_success_action=3, 
		@on_success_step_id=0, 
		@on_fail_action=3, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'SSIS', 
		@command=@cmd, 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [Call MT06 - CleanupOldBackup]    Script Date: 22/08/2025 7:32:17 AM ******/
SET @cmd = 
    N'/ISSERVER "\"\SSISDB\Maintenance\Gennovas.MaintenanceTask\MT06 - CleanupOldBackup.dtsx\"" ' +
    N'/SERVER "\"' + @serverName + N'\"" ' +
    N'/ENVREFERENCE 1 ' +
    N'/Par "\"$ServerOption::LOGGING_LEVEL(Int16)\"";1 ' +
    N'/Par "\"$ServerOption::SYNCHRONIZED(Boolean)\"";True ' +
    N'/CALLERINFO SQLAGENT /REPORTING E';
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Call MT06 - CleanupOldBackup', 
		@step_id=3, 
		@cmdexec_success_code=0, 
		@on_success_action=3, 
		@on_success_step_id=0, 
		@on_fail_action=3, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'SSIS', 
		@command=@cmd, 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [Call MT07 - CheckStorage]    Script Date: 22/08/2025 7:32:17 AM ******/
SET @cmd = 
    N'/ISSERVER "\"\SSISDB\Maintenance\Gennovas.MaintenanceTask\MT07 - CheckStorage.dtsx\"" ' +
    N'/SERVER "\"' + @serverName + N'\"" ' +
    N'/ENVREFERENCE 1 ' +
    N'/Par "\"$ServerOption::LOGGING_LEVEL(Int16)\"";1 ' +
    N'/Par "\"$ServerOption::SYNCHRONIZED(Boolean)\"";True ' +
    N'/CALLERINFO SQLAGENT /REPORTING E';
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Call MT07 - CheckStorage', 
		@step_id=4, 
		@cmdexec_success_code=0, 
		@on_success_action=3, 
		@on_success_step_id=0, 
		@on_fail_action=3, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'SSIS', 
		@command=@cmd, 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
/****** Object:  Step [Call MT08 - ReleaseMemory]    Script Date: 22/08/2025 7:32:17 AM ******/
SET @cmd = 
    N'/ISSERVER "\"\SSISDB\Maintenance\Gennovas.MaintenanceTask\MT08 - ReleaseMemory.dtsx\"" ' +
    N'/SERVER "\"' + @serverName + N'\"" ' +
    N'/ENVREFERENCE 1 ' +
    N'/Par "\"$ServerOption::LOGGING_LEVEL(Int16)\"";1 ' +
    N'/Par "\"$ServerOption::SYNCHRONIZED(Boolean)\"";True ' +
    N'/CALLERINFO SQLAGENT /REPORTING E';
EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Call MT08 - ReleaseMemory', 
		@step_id=5, 
		@cmdexec_success_code=0, 
		@on_success_action=1, 
		@on_success_step_id=0, 
		@on_fail_action=2, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'SSIS', 
		@command=@cmd, 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobschedule @job_id=@jobId, @name=N'Maintenance - Daily Tasks', 
		@enabled=1, 
		@freq_type=4, 
		@freq_interval=1, 
		@freq_subday_type=1, 
		@freq_subday_interval=0, 
		@freq_relative_interval=0, 
		@freq_recurrence_factor=0, 
		@active_start_date=20250820, 
		@active_end_date=99991231, 
		@active_start_time=100, 
		@active_end_time=235959, 
		@schedule_uid=N'fa302dbe-42d8-4c24-988f-6acf67edc492'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
COMMIT TRANSACTION
GOTO EndSave
QuitWithRollback:
    IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION
EndSave:
GO
```
