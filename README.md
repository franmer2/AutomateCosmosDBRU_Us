# Azure Cosmos DB: Manage RU with Azure Automation

In the following scenario, we need to ingest and process a large amount of information during a specific period of the day.
Since this treatment is done in a recurrence and predictable way, it is easy to plan the change of RU with a service like Azure automation.

We'll see in detail how to implement a solution with Azure Automation to make these RU changes based on our needs

## Prerequisite

- [Azure Free trial](https://azure.microsoft.com/en-us/free/)


# Creating Azure services
## Creating a resource group
We will start by creating a group of resources to host the different services of our solution.

From [Azure portal](https://portal.azure.com), click "**Create a resource**"

![sparkle](Pictures/001.png)

Then "**Resource group**"

 ![sparkle](Pictures/002.png)


Click "**Create**"

![sparkle](Pictures/003.png)

Name the resource group and click the button  "**Review + create**"

![sparkle](Pictures/004.png)

Then validate the creation by clicking the button "**Create**"

![sparkle](Pictures/005.png)

## Azure Cosmos DB account creation

From [Azure portal](https://portal.azure.com), click "**Create a resource**"

![sparkle](Pictures/006.png)

Search for Azure Cosmos DB

![sparkle](Pictures/007.png)

Then click "**Create**"

![sparkle](Pictures/008.png)

Select the previously created resource group, set the options you need, then click the button "**Review + create**"

![sparkle](Pictures/009.png)

Click "**Create**"

![sparkle](Pictures/010.png)

Once the Azure Cosmos DB account is created, click "**Go to resource**" to create database and container

![sparkle](Pictures/011.png)

Click "**Overview**" then "**Add Container**"

![sparkle](Pictures/012.png)

Create a new database and a new container. In this example we're going to uncheck the box "**Provision database throughput**" (but the script will also work with this checked box).

Click "**Ok**"

![sparkle](Pictures/013.png)

Once the base and container are created, they will be visible from your Azure Cosmos DB account overview 

![sparkle](Pictures/014.png)

## Azure Automation service creation

From [Azure portal ](https://portal.azure.com), click "**Create a resource**"

![sparkle](Pictures/015.png)


Then search for "**Automation**"

![sparkle](Pictures/016.png)

Click "**Create**

![sparkle](Pictures/017.png)

Name your Automation account, choose the resource group created previously and select "**Yes**" for "**Create Azure Run As account**"

Click "**Create**"

![sparkle](Pictures/018.png)

After creating the Azure Automation account, you should have the following resources in your resource group

![sparkle](Pictures/019.png)

## Azure Automation account setting up

Click on your **Azure Automation** account

![sparkle](Pictures/020.png)

Click on "**Runbooks**" then on "**Create a runbook**"

![sparkle](Pictures/021.png)

Name your runbook.

Select "**PowerShell**" in "**Runbook type**" drop-down list

Click "**Create**"

![sparkle](Pictures/022.png)


Copy the PowerShell script below. This script is also available on [Hugo Girard's Github repo](https://github.com/hugogirard/azureScripts/tree/master/runbook/scaleUnitCosmosDB), who helped a lot by writting the script. Thank you Hugo :)! 

```javascript

[cmdletbinding()]
param(
    [Parameter(Mandatory=$true)]
    [string]$resourceGroup,    
    [Parameter(Mandatory=$true)]
    [string]$cosmosdbAccount,
    [Parameter(Mandatory=$true)]
    [string]$database,    
    [string]$container,    
    [int]$databaseThroughput, # 1 for database throughput otherwise no need this parameters    
    [Parameter(Mandatory=$true)]
    [int]$newRUs
)
$connectionName = "AzureRunAsConnection"
try {
    $servicePrincipalConnection=Get-AutomationConnection -Name $connectionName
    "Logging in Azure..."
    Connect-AzAccount -ServicePrincipal `
        -TenantId $servicePrincipalConnection.TenantId `
        -ApplicationId $servicePrincipalConnection.ApplicationId `
        -CertificateThumbprint $servicePrincipalConnection.CertificateThumbprint
    if ($databaseThroughput -eq 1) {
        Write-Output "Update request units database level"
        $throughput = Get-AzCosmosDBSqlDatabaseThroughput -ResourceGroupName $resourceGroup `
                      -AccountName $cosmosdbAccount -Name $database
    } else {
        Write-Output "Update request units container level"
        $throughput = Get-AzCosmosDBSqlContainerThroughput -ResourceGroupName $resourceGroup `
        -AccountName $cosmosdbAccount -DatabaseName $database -Name $container
    }
    
    $currentRUs = $throughput.Throughput
    $minimumRUs = $throughput.MinimumThroughput
    
    Write-Output "Current throughput is $currentRUs. Minimum allowed throughput is $minimumRUs."
    
    if ([int]$newRUs -lt [int]$minimumRUs) {
        Write-Output "Requested new throughput of $newRUs is less than minimum allowed throughput of $minimumRUs."
        Write-Output "Using minimum allowed throughput of $minimumRUs instead."
        $newRUs = $minimumRUs
    }
    
    if ([int]$newRUs -eq [int]$currentRUs) {
        Write-Output "New throughput is the same as current throughput. No change needed."
    }
    else {
        Write-Output "Updating throughput to $newRUs."
        
        if ($databaseThroughput -eq 1) {
            Update-AzCosmosDBSqlDatabaseThroughput -ResourceGroupName $resourceGroup `
            -AccountName $cosmosdbAccount -Name $database `
            -Throughput $newRUs
        } else {
            Update-AzCosmosDBSqlContainerThroughput -ResourceGroupName $resourceGroup `
            -AccountName $cosmosdbAccount -DatabaseName $database `
            -Name $container -Throughput $newRUs
        }
    }
}
catch {
    if (!$servicePrincipalConnection)
    {
        $ErrorMessage = "Connection $connectionName not found."
        throw $ErrorMessage
    } else{
        Write-Error -Message $_.Exception
        throw $_.Exception
    }    
}
 ```


Click "**Save**"

![sparkle](Pictures/023.png)

At this point in the article, the script will not work yet. It is necessary to add two modules to make it work.

Click on the cross at the top right to close the editing window

![sparkle](Pictures/024.png)

Then do the same to close the runbook

![sparkle](Pictures/025.png)

On the left, click  "**Module**", then on "**Browse gallery**"

![sparkle](Pictures/026.png)

Add the following two modules: 

- Az.Accounts
- Az.CosmosDB

Below is an illustration for the Az.Accounts module

Search Az.Accounts module

![sparkle](Pictures/027.png)

Then click "**Import**" then in the next window on "**Ok**"

![sparkle](Pictures/028.png)

Repeat the operation for the Az.CosmosDB module

Make sure the modules are present and that the import is complete


![sparkle](Pictures/029.png)

Get back in your runbook

![sparkle](Pictures/030.png)

Click "**Edit**"

![sparkle](Pictures/031.png)

Click "**Test pane**"

![sparkle](Pictures/032.png)

1. Fill the fields on the left with the requested information regarding Azure Cosmos DB (here we set the RUs at the container level, so I left the field 'databasethroughput' empty, otherwise, to act at the database level, just enter the value: 1). I set the new RU level to 500 (instead of 400)..

2. Click "**Start**"

![sparkle](Pictures/033.png)

If everything goes right, you need to get the following message

![sparkle](Pictures/034.png)

But most importantly, your container now runs at 500 RU


![sparkle](Pictures/035.png)

## Schedule a recurring execution

Now that the hardest part is done, all we have to do is plan the execution of our script

From the runbook's editing window, click "**Publish**" and then "**Yes**"

![sparkle](Pictures/036.png)

In your runbook, click "**Schedules**"

![sparkle](Pictures/037.png)

Click "**Add a schedule**"

![sparkle](Pictures/038.png)

Click "**Schedule**" 

![sparkle](Pictures/039.png)

Puis "**Create a new schedule**"

![sparkle](Pictures/040.png)

Enter your planning parameters and then Click "**Create**"

![sparkle](Pictures/041.png)


 Click "**Configure parameters and run settings**"


![sparkle](Pictures/042.png)


Enter information about your Azure Cosmos DB container and then Click "**Ok**

![sparkle](Pictures/043.png)

Validate your schedule by clicking on "**Ok**"

![sparkle](Pictures/044.png)

Your schedule is now ready.

![sparkle](Pictures/045.png)

It is possible to create several other schedules, for example to decrease RU if necessary


![sparkle](Pictures/046.png)

## Monitoring

Click "**Jobs**" if you want to have a status of your schedules

![sparkle](Pictures/047.png)