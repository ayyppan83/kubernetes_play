To install azure cli (az)

https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest&pivots=msi-powershell

<img width="1272" height="235" alt="image" src="https://github.com/user-attachments/assets/94fc265d-82da-44ec-8b5a-43d766c2335e" />

## Install Azure CLI on Windows Using PowerShell

Run the following commands in an **elevated PowerShell window (Run as Administrator)**:

```powershell
$ProgressPreference = 'SilentlyContinue'

Invoke-WebRequest -Uri "https://aka.ms/installazurecliwindowsx64" -OutFile ".\AzureCLI.msi"

Start-Process msiexec.exe -Wait -ArgumentList '/I', 'AzureCLI.msi', '/quiet'

Remove-Item ".\AzureCLI.msi"
```

Verify the installation:

```powershell
az version
```

Login to Azure:

```powershell
 az login --use-device-code
```



<img width="1341" height="404" alt="image" src="https://github.com/user-attachments/assets/0f543b0c-cd06-4282-9361-80e0291aa93c" />

To list Azure subscriptions using Azure CLI:

```bash
az account list --output table
```

Show current active subscription

```
az account show --output table
```



<img width="668" height="215" alt="image" src="https://github.com/user-attachments/assets/1b32d348-7bd6-473d-9ccf-3523b5243a4a" />

<img width="1035" height="163" alt="image" src="https://github.com/user-attachments/assets/f92b6d80-2785-42ec-8ad3-28e8e0104442" />



$helmVersion = "v3.19.0"
Invoke-WebRequest -Uri "https://get.helm.sh/helm-$helmVersion-windows-amd64.zip" -OutFile "$env:TEMP\helm.zip"
Expand-Archive -Path "$env:TEMP\helm.zip" -DestinationPath "$env:TEMP\helm" -Force
Copy-Item "$env:TEMP\helm\windows-amd64\helm.exe" "C:\Windows\System32\helm.exe"
helm version

<img width="1106" height="167" alt="image" src="https://github.com/user-attachments/assets/ea91a40c-b0cc-4cd4-8964-a2dd626351f4" />



<img width="1071" height="343" alt="image" src="https://github.com/user-attachments/assets/77a46a7b-07b5-4bb4-93d3-cf0ddb95807a" />


<img width="798" height="76" alt="image" src="https://github.com/user-attachments/assets/68025098-eeb6-4899-a9d2-ce852d4eb409" />


