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
