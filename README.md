# Windows 11 Pro VM mit NVIDIA GPU Passthrough (Hyper-V)

## TPM Aktivieren in der VM

1. Rechtsklick auf die VM → **Einstellungen**
2. **Sicherheit** auswählen → **TPM aktivieren**

## Netzwerk-Setup umgehen

1. Bei der Netzwerkeinrichtung in der Windows installation: `Shift + F10` drücken
2. Im Terminal eingeben:

```cmd
oobe\bypassnro
```

3. VM startet neu → Installation fortsetzen
4. Netzwerkadapter auf **Default Switch** stellen:
   - Rechtsklick auf VM → **Einstellungen** → **Netzwerk**

## NVIDIA Treiber kopieren (Host-PC)

PowerShell als Administrator auf deinem PC ausführen ("Windows 11" mit dem Namen deiner VM ersetzen):

```powershell
$vm = "Windows 11"
$systemPath = "C:\Windows\System32\"
$driverPath = "C:\Windows\System32\DriverStore\FileRepository\"

$currentPrincipal = New-Object Security.Principal.WindowsPrincipal([Security.Principal.WindowsIdentity]::GetCurrent())
if($currentPrincipal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)){

    Get-VM -Name $vm | Get-VMIntegrationService | ? {-not($_.Enabled)} | Enable-VMIntegrationService -Verbose

    $localDriverFolder = ""
    Get-ChildItem $driverPath -recurse | Where-Object {$_.PSIsContainer -eq $true -and $_.Name -match "nv_dispi.inf_amd64_*"} | Sort-Object -Descending -Property LastWriteTime | select -First 1 |
    ForEach-Object {
        if ($localDriverFolder -eq "") {
            $localDriverFolder = $_.Name                                 
        }
    }

    Write-Host $localDriverFolder

    Get-ChildItem "$driverPath$localDriverFolder" -recurse | Where-Object {$_.PSIsContainer -eq $false} |
    ForEach-Object {
        $sourcePath = $_.FullName
        $destinationPath = $sourcePath -replace "^C\:\\Windows\\System32\\DriverStore\\","C:\Temp\System32\HostDriverStore\"
        Copy-VMFile $vm -SourcePath $sourcePath -DestinationPath $destinationPath -Force -CreateFullPath -FileSource Host
    }

    Get-ChildItem $systemPath | Where-Object {$_.Name -like "NV*"} |
    ForEach-Object {
        $sourcePath = $_.FullName
        $destinationPath = $sourcePath -replace "^C\:\\Windows\\System32\\","C:\Temp\System32\"
        Copy-VMFile $vm -SourcePath $sourcePath -DestinationPath $destinationPath -Force -CreateFullPath -FileSource Host
    }

    Write-Host "Success! Bitte gehe zu C:\Temp und kopiere die Dateien innerhalb der VM an den vorgesehenen Ort."
}else{
    Write-Host "Dieses Script muss als Admin ausgeführt werden."
}
```

> **Warten, bis "Success" angezeigt wird!**

## Innerhalb der VM: Dateien kopieren

1. Gehe zu `C:\Temp\System32`, alle Dateien kopieren
2. Einfügen in: `C:\Windows\System32`
3. VM ausschalten

## GPU Partition aktivieren (Host-PC)

PowerShell erneut als Admin ausführen:

```powershell
$vm = "Windows 11"
Remove-VMGpuPartitionAdapter -VMName $vm
Add-VMGpuPartitionAdapter -VMName $vm
Set-VMGpuPartitionAdapter -VMName $vm -MinPartitionVRAM 1
Set-VMGpuPartitionAdapter -VMName $vm -MaxPartitionVRAM 11
Set-VMGpuPartitionAdapter -VMName $vm -OptimalPartitionVRAM 10
Set-VMGpuPartitionAdapter -VMName $vm -MinPartitionEncode 1
Set-VMGpuPartitionAdapter -VMName $vm -MaxPartitionEncode 11
Set-VMGpuPartitionAdapter -VMName $vm -OptimalPartitionEncode 10
Set-VMGpuPartitionAdapter -VMName $vm -MinPartitionDecode 1
Set-VMGpuPartitionAdapter -VMName $vm -MaxPartitionDecode 11
Set-VMGpuPartitionAdapter -VMName $vm -OptimalPartitionDecode 10
Set-VMGpuPartitionAdapter -VMName $vm -MinPartitionCompute 1
Set-VMGpuPartitionAdapter -VMName $vm -MaxPartitionCompute 11
Set-VMGpuPartitionAdapter -VMName $vm -OptimalPartitionCompute 10
Set-VM -GuestControlledCacheTypes $true -VMName $vm
Set-VM -LowMemoryMappedIoSpace 1Gb -VMName $vm
Set-VM -HighMemoryMappedIoSpace 32GB -VMName $vm
Start-VM -Name $vm
```

> Nach **ENTER** startet die VM automatisch.

## Finaler Schritt

Wenn VM startet:
- Rechtsklick → **Verbinden**
- Im Gerätemanager prüfen, ob NVIDIA GPU erkannt wurde.

