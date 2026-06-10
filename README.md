# SystemWriteMonitor

```Powershell
function SystemWriteMonitor {
    $diskWrite = New-Object System.Diagnostics.PerformanceCounter("LogicalDisk", "Disk Write Bytes/sec", "_Total")
    $diskRead  = New-Object System.Diagnostics.PerformanceCounter("LogicalDisk", "Disk Read Bytes/sec", "_Total")
    $cpu       = New-Object System.Diagnostics.PerformanceCounter("Processor", "% Processor Time", "_Total")
 
    $diskWrite.NextValue() | Out-Null
    $diskRead.NextValue()  | Out-Null
    $cpu.NextValue()       | Out-Null
    Start-Sleep 1
 
    $global:FileLog = [System.Collections.Concurrent.ConcurrentQueue[string]]::new()
    $csvPath = Join-Path $env:USERPROFILE "Desktop\disk_activity.csv"
    if (-not (Test-Path $csvPath)) {
        "Date,Heure,Type,Taille_MB,Chemin" | Out-File $csvPath -Encoding UTF8
    }
    $watchPaths = @("C:\Users", "C:\Windows\Temp", "C:\Program Files", "C:\Program Files (x86)", "C:\ProgramData", "C:\Windows")
    $watchers   = @()
 
    foreach ($wp in $watchPaths) {
        if (-not (Test-Path $wp)) { continue }
        $w = New-Object System.IO.FileSystemWatcher
        $w.Path                  = $wp
        $w.IncludeSubdirectories = $true
        $w.NotifyFilter          = [System.IO.NotifyFilters]::FileName -bor
                                   [System.IO.NotifyFilters]::LastWrite -bor
                                   [System.IO.NotifyFilters]::Size
        $w.EnableRaisingEvents   = $true
 
        Register-ObjectEvent $w "Created" -Action {
            $path = $Event.SourceEventArgs.FullPath
            try {
                $item = Get-Item $path -ErrorAction Stop
                if (-not $item.PSIsContainer) {
                    $sizeMB = [math]::Round($item.Length / 1MB, 2)
                    if ($sizeMB -gt 0.01) {
                        $ts = Get-Date -f 'yyyy-MM-dd HH:mm:ss'
                        $global:FileLog.Enqueue(("[CREE]    {0}  {1,8:N2} MB  {2}" -f $ts, $sizeMB, $path))
                        "{0},{1},CREE,{2},{3}" -f (Get-Date -f 'yyyy-MM-dd'), (Get-Date -f 'HH:mm:ss'), $sizeMB, $path |
                            Out-File $csvPath -Append -Encoding UTF8
                    }
                }
            } catch {
                $global:FileLog.Enqueue(("[CREE]    {0}  -        MB  {1}" -f (Get-Date -f 'yyyy-MM-dd HH:mm:ss'), $path))
                "{0},{1},CREE,-,{2}" -f (Get-Date -f 'yyyy-MM-dd'), (Get-Date -f 'HH:mm:ss'), $path |
                    Out-File $csvPath -Append -Encoding UTF8
            }
        } | Out-Null
 
        Register-ObjectEvent $w "Changed" -Action {
            $path = $Event.SourceEventArgs.FullPath
            try {
                $item = Get-Item $path -ErrorAction Stop
                if (-not $item.PSIsContainer) {
                    $sizeMB = [math]::Round($item.Length / 1MB, 2)
                    if ($sizeMB -gt 0.5) {
                        $ts = Get-Date -f 'yyyy-MM-dd HH:mm:ss'
                        $global:FileLog.Enqueue(("[MODIFIE] {0}  {1,8:N2} MB  {2}" -f $ts, $sizeMB, $path))
                        "{0},{1},MODIFIE,{2},{3}" -f (Get-Date -f 'yyyy-MM-dd'), (Get-Date -f 'HH:mm:ss'), $sizeMB, $path |
                            Out-File $csvPath -Append -Encoding UTF8
                    }
                }
            } catch { }
        } | Out-Null
 
        $watchers += $w
    }
 
    $history = [System.Collections.Generic.LinkedList[string]]::new()
 
    try {
        while ($true) {
            Clear-Host
 
            $write    = $diskWrite.NextValue() / 1MB
            $read     = $diskRead.NextValue()  / 1MB
            $cpuV     = $cpu.NextValue()
            $mem      = Get-CimInstance Win32_OperatingSystem
            $usedMem  = ($mem.TotalVisibleMemorySize - $mem.FreePhysicalMemory) / 1MB
            $totalMem = $mem.TotalVisibleMemorySize / 1MB
 
            $top = Get-Process | Sort-Object CPU -Descending | Select-Object -First 8
 
            $diskProcs = Get-CimInstance Win32_Process |
                Select-Object ProcessId, Name, ExecutablePath,
                    @{N="WriteOpDelta"; E={ $_.WriteOperationCount }} |
                Sort-Object WriteOpDelta -Descending |
                Select-Object -First 6
 
            $line = ""
            while ($global:FileLog.TryDequeue([ref]$line)) {
                $history.AddLast($line) | Out-Null
                if ($history.Count -gt 10) { $history.RemoveFirst() }
            }
 
            Write-Host "===== MINI TASK MANAGER =====" -ForegroundColor Cyan
            Write-Host ""
 
            Write-Host "---- SYSTEME ----" -ForegroundColor DarkCyan
            Write-Host ("Disk Write : {0:N2} MB/s" -f $write) -ForegroundColor Yellow
            Write-Host ("Disk Read  : {0:N2} MB/s" -f $read)  -ForegroundColor Green
            Write-Host ("CPU        : {0:N1} %"    -f $cpuV)  -ForegroundColor Magenta
            Write-Host ("RAM        : {0:N1}/{1:N1} GB" -f ($usedMem/1024), ($totalMem/1024)) -ForegroundColor Cyan
            Write-Host ""
 
            Write-Host "---- TOP PROCESSUS (CPU) ----" -ForegroundColor DarkYellow
            foreach ($p in $top) {
                "{0,-25} CPU: {1,6:N1}" -f $p.ProcessName, $p.CPU | Write-Host
            }
            Write-Host ""
 
            Write-Host "---- ECRITURES DISQUE (cumul I/O) ----" -ForegroundColor DarkRed
            foreach ($p in $diskProcs) {
                $exePath = if ($p.ExecutablePath) { $p.ExecutablePath } else { "(inconnu)" }
                $short   = if ($exePath.Length -gt 55) { "..." + $exePath.Substring($exePath.Length - 52) } else { $exePath }
                Write-Host ("{0,-22} PID:{1,6}  Writes:{2,8}  {3}" -f `
                    $p.Name, $p.ProcessId, $p.WriteOpDelta, $short) -ForegroundColor Yellow
            }
            Write-Host ""
 
            Write-Host "---- FICHIERS CREES / MODIFIES (temps reel - plus recent en haut) ----" -ForegroundColor DarkMagenta
            if ($history.Count -eq 0) {
                Write-Host "  (en attente d'activite...)" -ForegroundColor DarkGray
            } else {
                foreach ($entry in ([System.Linq.Enumerable]::Reverse($history))) {
                    $color = if ($entry -like "*CREE*") { "Red" } else { "Magenta" }
                    Write-Host $entry -ForegroundColor $color
                }
            }
 
            Write-Host ""
            Write-Host ("Surveillance : {0}" -f ($watchPaths -join " | ")) -ForegroundColor DarkGray
            Write-Host ("Log          : {0}" -f $csvPath) -ForegroundColor DarkGray
            Write-Host "CTRL+C pour quitter" -ForegroundColor DarkGray
            Start-Sleep 2
        }
    } finally {
        $diskWrite.Dispose()
        $diskRead.Dispose()
        $cpu.Dispose()
        foreach ($w in $watchers) { $w.Dispose() }
        Get-EventSubscriber | Unregister-Event -ErrorAction SilentlyContinue
        Remove-Variable -Name FileLog -Scope Global -ErrorAction SilentlyContinue
    }
}
 
SystemWriteMonitor


```
