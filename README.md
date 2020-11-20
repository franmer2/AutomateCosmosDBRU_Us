# Azure Cosmos DB : Pilotez les RU avec Azure Automation

Dans le scénario suivant, nous avons besoin d'ingérer et traiter une grande quantité d'information durant une période précise de la journée.
Etant donné que ce traitement se fait de manière récurrence et prédictible, il est donc aisé de planifier le changement de RU avec un service comme Azure automation.

Nous allons voir en détail comment mettre en place une solution avec Azure Automation pour effectuer ces changements de RU en fonction de nos besoins

## Prérequis

- [Un abonnement Azure](https://azure.microsoft.com/fr-fr/free/)


# Création des services Azure
## Création d'un groupe de ressources
Nous allons commencer par créer un groupe de ressources afin d'héberger les différents services de notre solution.

Depuis le portail [Azure](https://portal.azure.com), cliquez sur "**Create a resource**"

![sparkle](Pictures/001.png)

 Puis, recherchez "**Resource group**"

 ![sparkle](Pictures/002.png)


Cliquez sur le bouton "**Create**"

![sparkle](Pictures/003.png)

Donnez un nom au groupe de ressources puis cliquez sur le bouton "**Review + create**"

![sparkle](Pictures/004.png)

Puis validez la création en cliquant sur le bouton "**Create**"

![sparkle](Pictures/005.png)

## Creation du compte Azure Cosmos DB

Depuis le portail [Azure](https://portal.azure.com), cliquez sur "**Create a resource**"

![sparkle](Pictures/006.png)

Recherchez le service Azure Cosmos DB

![sparkle](Pictures/007.png)

Puis cliquez sur le bouton "**Create**"

![sparkle](Pictures/008.png)

Sélectionnez le groupe de ressources créé précédemment, définissez les options dont vous avez besoin puis cliquez sur le bouton "**Review + create**"

![sparkle](Pictures/009.png)

Cliquez sur le bouton "**Create**"

![sparkle](Pictures/010.png)

Une fois le compte Azure Cosmos DB créé, cliquez sur le bouton "**Go to resource**" afin de créer une base et un conteneur.

![sparkle](Pictures/011.png)

Cliquez sur "**Overview**" puis sur "**Add Container**"

![sparkle](Pictures/012.png)

Créez une nouvelle base de données ainsi qu'un nouveau conteneur. Dans cet exemple nous allons décocher la case "**Provision database throughput**" (mais le script fonctionnera aussi avec cette case cochée).

Cliquez sur le bouton "**Ok**"

![sparkle](Pictures/013.png)

Une fois la base et le conteneur créés, ils devront être visibles depuis la vue d'ensemble de votre compte Azure Cosmos DB 

![sparkle](Pictures/014.png)

## Création du Service Azure Automation

Depuis le portail [Azure](https://portal.azure.com), cliquez sur "**Create a resource**"

![sparkle](Pictures/015.png)


Puis recherchez "**Automation**"

![sparkle](Pictures/016.png)

Cliquez sur le bouton "**Create**

![sparkle](Pictures/017.png)

Donnez un nom à votre compte "*Automation*", choisissez le groupe de ressources créé précédemment et sélectionnez "**Yes**" pour "**Create Azure Run As account**"

Cliquez sur le bouton "**Create**"

![sparkle](Pictures/018.png)

Après la création du compte "*Azure Automation*", vous devriez donc avoir les ressources suivantes dans votre groupe de ressources

![sparkle](Pictures/019.png)

## Paramétrage du compte Azure Automation

Cliquez sur votre compte **Azure Automation**

![sparkle](Pictures/020.png)

Cliquez sur "**Runbooks**" puis sur "**Create a runbook**"

![sparkle](Pictures/021.png)

Donnez un nom à votre *runbook*.
Choisissez "**PowerShell**" dans la liste déroulante "**Runbook type**"

Cliquez sur le bouton "**Create**"

![sparkle](Pictures/022.png)


Copiez le script PowerShell ci-dessous. Ce script est aussi disponible sur le [repo GitHub de Hugo Girard](https://github.com/hugogirard/azureScripts/tree/master/runbook/scaleUnitCosmosDB), qui a grandement contribué à l'écriture du script. Merci Hugo :) ! 

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


Puis cliquez sur le bouton "**Save**"

![sparkle](Pictures/023.png)

A ce stade-ci de l'article, le script ne fonctionnera pas encore. Il est nécessaire de rajouter deux modules pour le faire fonctionner.

Cliquez sur la croix en haut à droite pour fermer la fenêtre d'édition

![sparkle](Pictures/024.png)

Puis faîte de même pour fermer le runbook

![sparkle](Pictures/025.png)

Sur la gauche, cliquez sur "**Module**", puis sur "**Browse gallery**"

![sparkle](Pictures/026.png)

Rajoutez les 2 modules suivants :

- Az.Accounts
- Az.CosmosDB

Ci-dessous une illustration pour le module Az.Accounts

Recherchez le module Az.Accounts

![sparkle](Pictures/027.png)

Puis cliquez sur "**Import**" puis dans la fenêtre suivante sur "**Ok**"

![sparkle](Pictures/028.png)

Recommencez l'opération pour le module Az.CosmosDB

Vérifiez que les modules soient bien présents et que l'importation est bien terminée


![sparkle](Pictures/029.png)

Revenez dans votre runbook

![sparkle](Pictures/030.png)

Cliquez sur le bouton "**Edit**"

![sparkle](Pictures/031.png)

Cliquez sur "**Test pane**"

![sparkle](Pictures/032.png)

1. Remplissez les champs sur la gauche avec les informations demandées concernant Azure Cosmos DB (ici nous pilotons les RU au niveau du conteneur, donc j'ai laissé le champ *databasethroughput* vide, sinon, pour agir au niveau de la base de données, il suffit d'entrer la valeur : 1). J'ai défini le nouveau niveau de RU à 500 (au lieu de 400).

2. Cliquez sur "**Start**"

![sparkle](Pictures/033.png)

Si tout se passe correctement, vous devez obtenir le message suivant

![sparkle](Pictures/034.png)

Mais surtout, votre conteneur à maintenant 500 RU


![sparkle](Pictures/035.png)

## Programmer une exécution récurrente

Maintenant que le plus dur est fait, il ne nous reste plus qu'à planifier l'exécution de notre script

Depuis la fenêtre d'édition du runbook, cliquez sur "**Publish**" puis sur "**Yes**"

![sparkle](Pictures/036.png)

Dans votre runbook, cliquez sur "**Schedules**"

![sparkle](Pictures/037.png)

Cliquez sur "**Add a schedule**"

![sparkle](Pictures/038.png)

Cliquez sur "**Schedule**" 

![sparkle](Pictures/039.png)

Puis "**Create a new schedule**"

![sparkle](Pictures/040.png)

Entrez les paramètres de votre planification puis cliquez sur "**Create**"

![sparkle](Pictures/041.png)


Ensuite cliquez sur "**Configure parameters and run settings**"


![sparkle](Pictures/042.png)


Entrez les informations concernant votre conteneur Azure Cosmos DB puis cliquez sur "**Ok**

![sparkle](Pictures/043.png)

Validez la création de votre planification en cliquant sur "**Ok**"

![sparkle](Pictures/044.png)

Vous avez donc une planification programmée.

![sparkle](Pictures/045.png)

Il est possible de créer plusieurs autres planifications, pour par exemple revoir les RU à la baisse si besoin

![sparkle](Pictures/046.png)

## Monitoring

Cliquez sur "**Jobs**" si vous souhaitez avoir un statut de vos planifications

![sparkle](Pictures/047.png)