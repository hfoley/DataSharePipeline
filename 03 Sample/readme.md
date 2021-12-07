# 03 Sample
This folder contains the files you'll need to create all the components for the solution including a sample set of data.  The only file you need to update is the paramfile01.json.  All the scripts use this file to drive the names, locations, ect. to build it out in your environment.  Make sure you have rights to create resources in your Azure subscription.  You can use existing resources for this solution.  The script will check for it's existence first, before creating it.  

## File List - These items will be created in your Azure subscription

Filename  | Description
------------- | -------------
paramfile01.json | Parameter file that needs to be updated prior to running scripts
01 - Create Resources DataShare.ps1  | Creates all the items in Azure subscription
02 - GrantRightsDataShare.ps1 | Will grant rights to MSI and setup get secret permissions for Azure Key Vault
03 - Create Pipeline Parts DataShare.ps1 | Will create the pipelines and related items in Synapse workspace
AKVLS.json | Json file for creation of the linked server pointing to ADLS Gen 2 (azstorage2 in paramfile01.json)
LinkedServiceAzureSQLDB.json | Json file tied to the creation of the linked server pointing to Azure SQL DB 
CustomerStorageLS.json | Json file tied to creation of dataset to pull metadata for tables to extract
DataLakeLS.json | Json file tied to creation of dataset to pull metadata for tables to load
DataLakeSinkDS.json | Json file tied to creation of dataset to land parquet files
DynamicStorageSrcDS.json | Json file tied to creation of dataset to the custom logging table in Azure SQL DB
DynamicDataPullPL.json | Json file tied to creation of dataset to dedicated sql pool tables
paramfile03.json | Json file tied to creation of dataset to point to ADLS parquet files to load


## Azure Asset List - These items will be created in your Azure subscription
1. Azure Resource Group
2. Azure Key Vault - key vault will be used to store storage credentials  
3. 2 - Azure Data Lake Gen 2 accounts - one account for system use with Synapse, one for data lake and location to land extracted parquet files 
4. Azure Synapse Workspace - workspace where pipelines will be created
5. Azure Synapse - Dynamic Data Pull pipeline - parameter driven pipeline to pull CSVs from and land into ADLS Gen 2 lake storage account


## Steps 
1. Download all the files locally or to storage account fileshare used for CLI (see https://hopefoley.com/2021/09/27/powershell-in-the-clouds/ for help setting up).  Keep all the files in one folder location.   
1. Update the paramfile03.json with the values you want to use for the rest of the scripts.  Storage is finicky in the rules it has for naming.  Keep storage params lowercase and under 15 characters.  You will need to replace any values containing <text>.  Anything without <> surrounding it is optional to change.  
2. Run the 01 - Create Resources DataShare.ps1 script and supply the param file location.  You'll be prompted for your login credentials to Azure.  You'll also be prompted for a username and password.  This will become your Synapse workspace SQL admin login and password.  Below is some sample syntax to run the file and pass the paramfile within Azure CLI and locally.  Keep all your scripts, paramfile03.json and all json files in the same folder location.  
  Azure CLI:  `./"01 - Create Resources DataShare.ps1" -filepath ./paramfile03.json`<br>
  Locally:  `& "C:\folder\01 - Create Resources DataShare.ps1" -filepath "C:\folder\paramfile03.json"`
3. Run the 02 - GrantRightsDataShare.ps1 script.  You'll again be prompted for login to Azure.  This script will assign the rights needed to the ADLS storage account.  It will grant your account (or the admin user provided in the paramfile) to the role Storage Blob Data Contributor role on the ADLS account.  Below is a sample syntax.  
  Azure CLI:  `./"02 - GrantRightsDataShare.ps1" -filepath ./paramfile03.json`<br>
  Locally:  `& "C:\folder\02 - GrantRightsDataShare.ps1" -filepath "C:\folder\paramfile03.json"`
4. Run the 03 - CreateSynLoadPipelineParts.ps1 script.  You'll again be prompted for login to Azure.  This script will create the items within the Synapse workspace to build the pipelines.  It will create linked services, datasets, and pipelines.  Below is a sample syntax.  
  Azure CLI:  `./"03 - Create Pipeline Parts DataShare.ps1" -filepath ./paramfile03.json`<br>
  Locally:  `& "C:\folder\03 - Create Pipeline Parts DataShare.ps1" -filepath "C:\folder\paramfile03.json"`
6. Navigate to the Synapse workspace and open up Synapse Studio.  Navigate to the manage pane (far left toolbox icon).  Select Linked Services and find the linked service for your Azure SQL DB.  Update the values required and supply your credentials and verify connectivity by hitting the Test Connection button (you'll need to enable the IR for this)
7. Connect to the Azure SQL DB to create the metadata tables.  Open the SampleMetadataCreate.sql file and update the insert statements for the extracts with your Azure SQL DB.  
8. Use the same Azure SQL DB connection and open the script CowBiometricsSampleSource.sql.  Run this script to create the COW.Biometrics table and insert sample data.  
8. Run the script to create and insert statements for [ADF].[ExtractTables].  You will come back later to the insert for [ADF].[MetadataLoad] once you have a file to load.  
9. Navigate to your dedicated SQL pool.  You can do this within Synapse Studio or use the dedicated SQL endpoint and your preferred query tool.  Make sure that your dedicated pool is running.  Open COWBiometricsSynapseTargetCreate.sql script and run it to create the destination tables we'll load with the parquet files.  
10.  Navigate to pipelines (left pane pipe icon).  Run the SQL Date Based Extract pipeline - this pipeline will extract data based on the date range in the [ADF].[ExtractTables].  The sample is set to extract all the rows in the COW.Biometrics table that have BirthDate column values between 2019-02-18' to '2019-02-22'.  
10.  Verify the parquet file extracts successfully into your ADLS storage location (paramfile01.json value supplied for azstoragename2).  You can skip to #12 if you'd like to upload it now. 
11.  Run the PL SQL Not Date Based Extract pipeline - this pipeline will extract data based on the [NotDateColumn] value in the [ADF].[ExtractTables].  You need to supply a valid column value for the NotDateColumnValue parameter.  The NotDateColumn needs to be the value the column contained in the metadata [NotDateColumn].  For example, the sample is set to extract all the rows in the COW.Biometrics table that have Animal column value = value you specify at run time.  You can use 9111 for first value and Animal for the second. That will extract all the rows where the Animal = 9111.  
12.  Now that we have parquet files to use, we can now update the [ADF].[MetadataLoad] table with filenames we want to load.  Navigate back to query tool and update the insert statement with filename from one of the extract pipeline runs above.  Run the insert statement.  
13.  You can now run the Truncate Load Synapse pipeline.  You will be prompted again for the filename that will match to our [ADF].[MetadataLoad] for where it will load the data in the dedicated SQL pool.  This pattern will truncate and reload the destination table.  Verify it loads successfully (can use script DemoWatchSynapseLoadTables.sql). 
14.  You can also now run the Incremental Load pipeline.  This will use a staging table to load what's contained in the parquet file.  It will then trunate the staging table, check for values in the final target table that match, delete them, and reload from the staging table.  
15.  Validate you have values within your [COW].[Biometrics_Stg] and [COW].[Biometrics] tables.  You can add another entry into the [ADF].[MetadataLoad] for the other extracted parquet file and re-run the pipeline passing that filename into the parameter.  
16. If running the SQL Not Date Based Extract, validate the logging is captured in ADF.PipelineLog table. Note: not all fields populate (update later to come to resolve)