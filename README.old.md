---
services: virtual machines
platforms: powershell
author: msonecode
---

# How to create Azure Virtual Machine (VM) by PowerShell using ARM API

## Introduction
This sample demonstrates [how to make use of  Azure ARM (Azure Resource Manager) cmdlet to create RM VM on Azure platform](https://gallery.technet.microsoft.com/How-to-create-Azure-VM-by-22f8bea9).

**Related Topics**
- [How to create Azure VM by Powershell using classic ASM API version](https://gallery.technet.microsoft.com/How-to-create-Azure-VM-by-b894d750)
- [How to set expired date of Azure Virtual Machine by PowerShell](https://gallery.technet.microsoft.com/How-to-set-expired-date-of-826800a7)

## Scenarios
Currently, there exist two kinds of API to manage Azure Resource. One is the latest API suite ARM (Azure Resource Management) API, the other is the old classic ASM (Azure Service Model) API. Both API suites can be used to create VM. However, only ARM API can create RM VM that can be viewed on the Azure Portal [https://portal.azure.com](https://portal.azure.com).

IT Pros might be required to automate the VM creation job through Powershell, yet they cannot find enough material to instruct them in going through the ARM API to create VM. This sample tries to amend the instruction.

## Prerequisite 
- Install Azure PowerShell according to [https://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/](https://azure.microsoft.com/en-us/documentation/articles/powershell-install-configure/)
- Persisting Azure PowerShell logins [https://blogs.msdn.microsoft.com/stuartleeks/2015/12/11/persisting-azure-powershell-logins/](https://blogs.msdn.microsoft.com/stuartleeks/2015/12/11/persisting-azure-powershell-logins/)

## Script
- **Open the script file** RMCreateVM.ps1
- **Edit the** parameters  ResourceGroupName,LocationName,VMName,RmProfilePath **and then save the file**  
```ps1
Create-AzureRmVM  -ResourceGroupName ocoslab-tanyue -LocationName eastasia -VMName vm-frta-test01 -RmProfilePath C:\PS\azureaccount.json
```
- **Open the PowerShell and run the script file** RMCreateVM.ps1  
![][1]  

**Here are some code snippets for your reference.**  

```ps1
# Create a Windows VM using Resource Manager 
function Create-AzureRmVM(){ 
     param 
    ( 
      [string] 
      $RmProfilePath =$(throw "Parameter missing: -RmProfilePath RmProfilePath"), 
      [string] 
      $ResourceGroupName =$(throw "Parameter missing: -ResourceGroupName ResourceGroupName"), 
      [string] 
      $LocationName =$(throw "Parameter missing: -LocationName LocationName"), 
      [string] 
      $VMName =$(throw "Parameter missing: -VMName VMName"), 
      [string] 
      $VMSizeName ="Standard_DS1", 
      [string] 
      $PublisherName = 'MicrosoftVisualStudio', 
      [string] 
      $OfferName = 'Windows', 
      [string] 
      $SkusName = '10-Enterprise-N', 
      [string] 
      $UserName = 'frank', 
      [string] 
      $Password = 'Frank@12345678' 
 
    ) 
    
    Try 
    { 
       #1. Login Azure by profile or Login-AzureRmAccount 
       #Login-AzureRmAccount 
       #Save-AzureRmProfile -Path “C:\PS\azureaccount.json” 
       Write-Host "Login Azure by profile" -ForegroundColor Green    
       Select-AzureRmProfile –Path $RmProfilePath -ErrorAction Stop 
 
       #2. Check location 
       if(Check-AzureRmLocation -LocationName $LocationName){ 
          #3. Check resource group, if not, created it. 
          if(Check-AzureRmResourceGroup -LocationName $LocationName -ResourceGroupName $ResourceGroupName){ 
             #4. Check VM images   
             Write-Host "Check VM images $SkusName" -ForegroundColor Green     
             If(Get-AzureRMVMImageSku -Location $LocationName -PublisherName $PublisherName -Offer $OfferName -ErrorAction Stop | Where-Object {$_.Skus -eq $SkusName}){ 
                 #5. Check VM 
                 If(Get-AzureRmVM -Name $VMName -ResourceGroupName $ResourceGroupName -ErrorAction Ignore){ 
                     Write-Host -ForegroundColor Red "VM $VMName has already exist." 
                 } 
                 else{ 
                    #6. Check VM Size 
                    Write-Host "check VM Size $VMSizeName" -ForegroundColor Green   
                    If(Get-AzureRmVMSize -Location $LocationName | Where-Object {$_.Name -eq $VMSizeName}) 
                    { 
                       #7. Create a storage account 
                      $BlobURL = AutoGenerate-AzureRmStorageAccount -Location $LocationName -ResourceGroupName $ResourceGroupName 
                      If($BlobURL){ 
                        #8. Create a network interface 
                        $Nid = AutoGenerate-AzureRmNetworkInterface -Location $LocationName -ResourceGroupName $ResourceGroupName -VMName $VMName 
                        If($Nid){ 
                            Write-Host "Creating VM $VMName ..." -ForegroundColor Green  
                              
                            #10.Set the administrator account name and password for the virtual machine. 
                            $StrPass = ConvertTo-SecureString -String $Password -AsPlainText -Force 
                            $Cred = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList ($UserName, $StrPass) 
 
                            #11.Choose virtual machine size, set computername and credential 
                            $VM = New-AzureRmVMConfig -VMName $VMName -VMSize $VMSizeName -ErrorAction Stop 
                            $VM = Set-AzureRmVMOperatingSystem -VM $VM -Windows -ComputerName $VMName -Credential $Cred -ProvisionVMAgent -EnableAutoUpdate -ErrorAction Stop 
                            
                            #12.Choose source image 
                            $VM = Set-AzureRmVMSourceImage -VM $VM -PublisherName $PublisherName -Offer $OfferName -Skus $SkusName -Version "latest" -ErrorAction Stop 
                            
                            #13.Add the network interface to the configuration. 
                            $VM = Add-AzureRmVMNetworkInterface -VM $VM -Id $Nid -ErrorAction Stop 
                            
                            #14.Add storage that the virtual hard disk will use.  
                            $BlobPath = "vhds/"+$SkusName+"Disk.vhd" 
                            $OSDiskUri = $BlobURL + $BlobPath 
                            $DiskName = "windowsvmosdisk" 
                            $VM = Set-AzureRmVMOSDisk -VM $VM -Name $DiskName -VhdUri $OSDiskUri -CreateOption fromImage -ErrorAction Stop 
 
                            #15. Create a virtual machine 
                            New-AzureRmVM -ResourceGroupName $ResourceGroupName -Location $LocationName -VM $VM -ErrorAction Stop 
                            Write-Host "Successfully created a virtual machine $VMName" -ForegroundColor Green   
                        } 
                      } 
                    } 
                    Else 
                    { 
                       Write-Host -ForegroundColor Red "VM Size $VMSizeName does nott exist." 
                    } 
                     
                 } 
             } 
              Else{ 
                 Write-Host -ForegroundColor Red "VM images does not exist." 
             } 
          } 
       } 
       
    } 
    Catch 
    { 
          Write-Host -ForegroundColor Red "Create a virtual machine $VMName failed" $_.Exception.Message 
          return $false 
    } 
}
```
## Additional Resources 
- Manage Azure resource: [https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-ps-manage/](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-ps-manage/)

[1]: images/1.png
