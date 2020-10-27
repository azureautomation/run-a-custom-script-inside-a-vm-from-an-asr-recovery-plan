Run a custom script inside a VM from an ASR Recovery Plan
=========================================================

            

 



PowerShell Workflow
Edit|Remove
powershellworkflow
<#
.SYNOPSIS
    Runbook to run a custom script inside an Azure Virtual Machine
    This script needs to be used along with recovery plans in Azure Site Recovery.

.DESCRIPTION
    This runbook provides a way to run a script inside the guest virtual machine using custom script extension
    This script is to be used with VMs that are not failed over by recovery plan but are required for the application to function in Azure
    For Example : Updating DNS by going inside DNS VM, Failing over sql availability group
     
.DEPENDENCIES
    Azure VM agent should be installed in the VM before it is executed
    If it is not already installed install it inside the VM from http://aka.ms/vmagentwin
    Script that needs to run inside the virtual machine should already be uploaded in a storage account
    Use the following commands to upload the script
    
    $context = New-AzureStorageContext -StorageAccountName 'ScriptStorageAccountName' -StorageAccountKey 'ScriptStorageAccountKey'
    Set-AzureStorageBlobContent -Blob 'script.ps1' -Container 'script-container' -File 'ScriptLocalFilePath' -context $context
   
    Make sure that you specify the name and cloud service name of the VM into which the script will be run
    $VMRoleName and $VMServiceName should be set to a string with the required value

    Make sure you specify the storage container name and the script name. 
    $ScriptContainer and $ScriptName should be set to a string with the required value

    Make sure you specify the aruments to be passed to the script in variable $Args
    Example: $Args = '-Argument1 value1 -Argument2 value2'                     


.ASSETS
    Add Assets 'ScriptStorageAccountName' and 'ScriptStorageAccountKey' in the azure automation account
    You can choose to encrtypt these assets
    ScriptStorageAccountName: Name of the storage account where the script is stored
    ScriptStorageAccountKey: Key for the storage account where the script is stored
   
.PARAMETER RecoveryPlanContext
    RecoveryPlanContext is the only parameter you need to define.
    This parameter gets the failover context from the recovery plan. 

.NOTES
	Author: Prateek Sharma - pratshar@microsoft.com 
	Last Updated: 29/04/2015   
#>

workflow RunScriptInVM
{
    param (
        [Object]$RecoveryPlanContext
    )

    $Cred = Get-AutomationPSCredential -Name 'AzureCredential'

    # Connect to Azure
    $AzureAccount = Add-AzureAccount -Credential $Cred
    $AzureSubscriptionName = Get-AutomationVariable –Name 'AzureSubscriptionName'
    Select-AzureSubscription -SubscriptionName $AzureSubscriptionName
    
    #Specify the VM onto which you want to act upon
        
    $VMRoleName = ''
    $VMServiceName = ''
    
    #Provide the storage account name and the storage account key information
    $StorageAccountName = Get-AutomationVariable –Name 'ScriptStorageAccountName'
    $StorageAccountKey  =  Get-AutomationVariable –Name 'ScriptStorageAccountKey'
    
    #Provide the script details
    $ScriptContainer = ''
    $ScriptName = ''
    
    InLineScript
    {
       $context = New-AzureStorageContext -StorageAccountName $Using:StorageAccountName `
                  -StorageAccountKey $Using:StorageAccountKey;
       $sasuri = New-AzureStorageBlobSASToken -Container $Using:ScriptContainer `
                 -Blob $Using:ScriptName -Permission r -FullUri -Context $context
       
       $AzureVM = Get-AzureVM -ServiceName $Using:VMServiceName -Name $Using:VMRoleName      
         
       Write-Output 'UnInstalling custom script extension'
       Set-AzureVMCustomScriptExtension -Uninstall -ReferenceName CustomScriptExtension -VM $AzureVM | Update-AzureVM
         
       Write-Output 'Installing custom script extension'
       Set-AzureVMExtension -ExtensionName CustomScriptExtension -VM $AzureVM -Publisher Microsoft.Compute -Version 1.4 | Update-AzureVM   
       
       Write-output 'Running script on the VM'
       Set-AzureVMCustomScriptExtension -VM $AzureVM -FileUri $sasuri -Run $Using:ScriptName | Update-AzureVM
       
       # If your script requires argument the comment the line above and use the following two lines
       #$Args = ''           
       #Set-AzureVMCustomScriptExtension -VM $AzureVM -FileUri $sasuri -Run $Using:ScriptName -Argument $Args | Update-AzureVM

       Write-output 'Completed running script on the VM'                
    }
    
}

<# 
.SYNOPSIS 
    Runbook to run a custom script inside an Azure Virtual Machine 
    This script needs to be used along with recovery plans in Azure Site Recovery. 
 
.DESCRIPTION 
    This runbook provides a way to run a script inside the guest virtual machine using custom script extension 
    This script is to be used with VMs that are not failed over by recovery plan but are required for the application to function in Azure 
    For Example : Updating DNS by going inside DNS VM, Failing over sql availability group 
      
.DEPENDENCIES 
    Azure VM agent should be installed in the VM before it is executed 
    If it is not already installed install it inside the VM from http://aka.ms/vmagentwin 
    Script that needs to run inside the virtual machine should already be uploaded in a storage account 
    Use the following commands to upload the script 
     
    $context = New-AzureStorageContext -StorageAccountName 'ScriptStorageAccountName' -StorageAccountKey 'ScriptStorageAccountKey' 
    Set-AzureStorageBlobContent -Blob 'script.ps1' -Container 'script-container' -File 'ScriptLocalFilePath' -context $context 
    
    Make sure that you specify the name and cloud service name of the VM into which the script will be run 
    $VMRoleName and $VMServiceName should be set to a string with the required value 
 
    Make sure you specify the storage container name and the script name.  
    $ScriptContainer and $ScriptName should be set to a string with the required value 
 
    Make sure you specify the aruments to be passed to the script in variable $Args 
    Example: $Args = '-Argument1 value1 -Argument2 value2'                      
 
 
.ASSETS 
    Add Assets 'ScriptStorageAccountName' and 'ScriptStorageAccountKey' in the azure automation account 
    You can choose to encrtypt these assets 
    ScriptStorageAccountName: Name of the storage account where the script is stored 
    ScriptStorageAccountKey: Key for the storage account where the script is stored 
    
.PARAMETER RecoveryPlanContext 
    RecoveryPlanContext is the only parameter you need to define. 
    This parameter gets the failover context from the recovery plan.  
 
.NOTES 
    Author: Prateek Sharma - pratshar@microsoft.com  
    Last Updated: 29/04/2015    
#> 
 
workflow RunScriptInVM 
{ 
    param ( 
        [Object]$RecoveryPlanContext 
    ) 
 
    $Cred = Get-AutomationPSCredential -Name 'AzureCredential' 
 
    # Connect to Azure 
    $AzureAccount = Add-AzureAccount -Credential $Cred 
    $AzureSubscriptionName = Get-AutomationVariable –Name 'AzureSubscriptionName' 
    Select-AzureSubscription -SubscriptionName $AzureSubscriptionName 
     
    #Specify the VM onto which you want to act upon 
         
    $VMRoleName = '' 
    $VMServiceName = '' 
     
    #Provide the storage account name and the storage account key information 
    $StorageAccountName = Get-AutomationVariable –Name 'ScriptStorageAccountName' 
    $StorageAccountKey  =  Get-AutomationVariable –Name 'ScriptStorageAccountKey' 
     
    #Provide the script details 
    $ScriptContainer = '' 
    $ScriptName = '' 
     
    InLineScript 
    { 
       $context = New-AzureStorageContext -StorageAccountName $Using:StorageAccountName ` 
                  -StorageAccountKey $Using:StorageAccountKey; 
       $sasuri = New-AzureStorageBlobSASToken -Container $Using:ScriptContainer ` 
                 -Blob $Using:ScriptName -Permission r -FullUri -Context $context 
        
       $AzureVM = Get-AzureVM -ServiceName $Using:VMServiceName -Name $Using:VMRoleName       
          
       Write-Output 'UnInstalling custom script extension' 
       Set-AzureVMCustomScriptExtension -Uninstall -ReferenceName CustomScriptExtension -VM $AzureVM | Update-AzureVM 
          
       Write-Output 'Installing custom script extension' 
       Set-AzureVMExtension -ExtensionName CustomScriptExtension -VM $AzureVM -Publisher Microsoft.Compute -Version 1.4 | Update-AzureVM    
        
       Write-output 'Running script on the VM' 
       Set-AzureVMCustomScriptExtension -VM $AzureVM -FileUri $sasuri -Run $Using:ScriptName | Update-AzureVM 
        
       # If your script requires argument the comment the line above and use the following two lines 
       #$Args = ''            
       #Set-AzureVMCustomScriptExtension -VM $AzureVM -FileUri $sasuri -Run $Using:ScriptName -Argument $Args | Update-AzureVM 
 
       Write-output 'Completed running script on the VM'                 
    } 
     
}




This runbook provides a way to run a script inside the guest virtual machine using custom script extension


This script is to be used with VMs that are not failed over by an ASR Recovery Plan but are requried for the application to function in Azure


Example : Updating DNS, Faling over SQL Always On Availability Group


The given script is only a sample script. Ensure that you have provided all the variables for the script to work.


        
    
TechNet gallery is retiring! This script was migrated from TechNet script center to GitHub by Microsoft Azure Automation product group. All the Script Center fields like Rating, RatingCount and DownloadCount have been carried over to Github as-is for the migrated scripts only. Note : The Script Center fields will not be applicable for the new repositories created in Github & hence those fields will not show up for new Github repositories.
