
Script interactif de gestion des horaires de connexion Active Directory.
A executer sur SRVWIN01 (DC) en tant qu'administrateur.

Emplacement recommande : C:\Scripts\Set-LogonHoursRestriction.ps1

```powershell
# Requires -Module ActiveDirectory
<#
.SYNOPSIS
    Script interactif universel de gestion des horaires de connexion AD.
    Navigation fluide avec retour, ecrans dynamiques, drill-down OUs.
.EXAMPLE
    .\Set-LogonHoursRestriction.ps1
#>

# ============================================================
# CONFIGURATION
# ============================================================
$LogDir = "C:\Logs\LogonHours"
$Timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
$LogFile = "$LogDir\LogonHours_$Timestamp.log"
$ReportCSV = "$LogDir\LogonHours_Report_$Timestamp.csv"
if (!(Test-Path $LogDir)) { New-Item -Path $LogDir -ItemType Directory -Force | Out-Null }

# ============================================================
# FONCTIONS UTILITAIRES
# ============================================================
function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    Add-Content -Path $LogFile -Value "[$Level] $(Get-Date -Format 'HH:mm:ss') - $Message"
}

function Write-Banner {
    param([string]$Text, [string]$Color = "Cyan")
    $l = "=" * 60
    Write-Host $l -ForegroundColor $Color
    Write-Host "  $Text" -ForegroundColor $Color
    Write-Host $l -ForegroundColor $Color
    Write-Host ""
}

function Write-Info {
    param([string]$Label, [string]$Value)
    Write-Host "  $Label : " -ForegroundColor Gray -NoNewline
    Write-Host $Value -ForegroundColor White
}

function Write-Option {
    param([string]$Key, [string]$Text)
    Write-Host "  [$Key] " -ForegroundColor Yellow -NoNewline
    Write-Host $Text -ForegroundColor White
}

function Read-Key {
    param([string]$Prompt = "  Votre choix : ", [string[]]$Valid)
    do {
        Write-Host $Prompt -ForegroundColor Cyan -NoNewline
        $r = Read-Host
        if ($r -in $Valid) { return $r }
        Write-Host "  Choix invalide." -ForegroundColor Red
    } while ($true)
}

function Read-Num {
    param([string]$Prompt, [int]$Min, [int]$Max, [switch]$AllowZero)
    do {
        Write-Host $Prompt -ForegroundColor Cyan -NoNewline
        $r = Read-Host
        if ($AllowZero -and $r -eq "0") { return 0 }
        $n = 0
        if ([int]::TryParse($r, [ref]$n) -and $n -ge $Min -and $n -le $Max) { return $n }
        Write-Host "  Nombre invalide ($Min-$Max)" -ForegroundColor Red
    } while ($true)
}

function Read-MultiSelect {
    param([string]$Prompt, [int]$Max)
    do {
        Write-Host $Prompt -ForegroundColor Cyan -NoNewline
        $r = (Read-Host).Trim()
        if ($r -eq "0") { return @(0) }
        if ($r -in @("*","all","tout")) { return @(1..$Max) }
        $sel = @()
        foreach ($p in ($r -split ',')) {
            $p = $p.Trim()
            if ($p -match '^(\d+)-(\d+)$') { $s=[int]$Matches[1]; $e=[int]$Matches[2]; if($s -ge 1 -and $e -le $Max -and $s -le $e){ $sel += @($s..$e) } }
            elseif ($p -match '^\d+$') { $n=[int]$p; if($n -ge 1 -and $n -le $Max){ $sel += $n } }
        }
        $sel = $sel | Sort-Object -Unique
        if ($sel.Count -gt 0) { return $sel }
        Write-Host "  Format invalide. Ex: 1,3,5 ou 1-5 ou tout ou 0=retour" -ForegroundColor Red
    } while ($true)
}

function Build-LogonHours {
    param([int]$Start, [int]$End, [bool[]]$DaysEnabled)
    $DA = ""; for ($h=0; $h -lt 24; $h++) { if ($h -ge $Start -and $h -lt $End) { $DA += "1" } else { $DA += "0" } }
    $DB = "0" * 24
    $AW = ""; for ($d=0; $d -lt 7; $d++) { if ($DaysEnabled[$d]) { $AW += $DA } else { $AW += $DB } }
    $UTC = (Get-TimeZone).BaseUtcOffset.TotalHours
    if ($UTC -lt 0) { $p1=$AW.Substring(0,168+$UTC); $p2=$AW.Substring(168+$UTC); $AW="$p2$p1" }
    elseif ($UTC -gt 0) { $p1=$AW.Substring(0,$UTC); $p2=$AW.Substring($UTC); $AW="$p2$p1" }
    [byte[]]$b = New-Object byte[] 21; $gs = $AW -split '(\d{8})' | Where-Object { $_ -match '(\d{8})' }; $i=0
    foreach ($g in $gs) { $c=$g.ToCharArray(); [array]::Reverse($c); $b[$i]=[Convert]::ToByte((-join $c),2); $i++ }
    return $b
}

function Get-RestrictionLabel {
    param([byte[]]$B)
    if ($null -eq $B -or $B.Length -eq 0) { return "Aucune" }
    if (($B | Where-Object {$_ -eq 255}).Count -eq 21) { return "Aucune" }
    if (($B | Where-Object {$_ -eq 0}).Count -eq 21) { return "Bloque 24h/24" }
    return "Active"
}

# ============================================================
# SCAN INITIAL
# ============================================================
Clear-Host
Write-Banner "SCAN DE L'ENVIRONNEMENT ACTIVE DIRECTORY"
Write-Host "  Analyse en cours..." -ForegroundColor Yellow

try { Import-Module ActiveDirectory -ErrorAction Stop } catch {
    Write-Host "  ERREUR : Module Active Directory non disponible." -ForegroundColor Red
    Write-Host "  Ce script doit etre execute sur un DC ou un poste avec RSAT." -ForegroundColor Red
    exit 1
}

$Forest = Get-ADForest; $Domain = Get-ADDomain
$DC = $env:LOGONSERVER -replace '\\\\'; if ([string]::IsNullOrEmpty($DC)) { $DC = (Get-ADDomainController).HostName }
$TZ = Get-TimeZone; $UTCOff = $TZ.BaseUtcOffset.TotalHours; $UTCSgn = if($UTCOff -ge 0){"+"}else{""}
$DCs = Get-ADDomainController -Filter *

$AllOUs = Get-ADOrganizationalUnit -Filter * -SearchBase $Domain.DistinguishedName | Where-Object { $_.Name -ne "Domain Controllers" } | Sort-Object Name
$script:OUData = @()

foreach ($ou in $AllOUs) {
    $allU = Get-ADUser -Filter * -SearchBase $ou.DistinguishedName -SearchScope OneLevel -Properties Enabled, AdminCount, logonHours
    $enabled = @($allU | Where-Object { $_.Enabled -eq $true })
    $disabled = @($allU | Where-Object { $_.Enabled -eq $false })
    $admins = @($enabled | Where-Object { $_.AdminCount -eq 1 -or $_.SamAccountName -in @("Administrator","Administrateur","krbtgt") })
    $standards = @($enabled | Where-Object { $_.AdminCount -ne 1 -and $_.SamAccountName -notin @("Administrator","Administrateur","krbtgt") })
    if ($allU.Count -gt 0) {
        $restricted = @($enabled | Where-Object { $null -ne $_.logonHours -and ($_.logonHours | Where-Object {$_ -ne 255}).Count -gt 0 }).Count
        $script:OUData += [PSCustomObject]@{
            Name=$ou.Name; DN=$ou.DistinguishedName
            Standards=$standards.Count; Admins=$admins.Count; Disabled=$disabled.Count
            UserList=$enabled; StandardList=$standards; AdminList=$admins
            HasRestriction=$restricted
        }
    }
}

$TotalStd = ($script:OUData | Measure-Object -Property Standards -Sum).Sum
$TotalAdm = ($script:OUData | Measure-Object -Property Admins -Sum).Sum
$TotalDis = ($script:OUData | Measure-Object -Property Disabled -Sum).Sum

Write-Host "  Scan termine.`n" -ForegroundColor Green
Write-Log "Scan: $($Domain.DNSRoot) | $($script:OUData.Count) OUs | $TotalStd std + $TotalAdm adm"

# ============================================================
# FONCTIONS D'AFFICHAGE
# ============================================================

function Show-EnvInfo {
    Write-Info "Date" (Get-Date -Format "dd/MM/yyyy")
    Write-Info "Heure" (Get-Date -Format "HH\Hmm")
    Write-Info "Foret" $Forest.Name
    Write-Info "Domaine" $Domain.DNSRoot
    Write-Info "DC" $DC
    Write-Info "Unites organisationnelles" "$($script:OUData.Count)"
    Write-Host ""
}

function Show-OUTable {
    Write-Host ("  {0,-5} {1,-38} {2,-8} {3,-7} {4,-10}" -f "#","OU","Users","Admin","Restreint") -ForegroundColor Gray
    Write-Host ("  {0,-5} {1,-38} {2,-8} {3,-7} {4,-10}" -f "-----","--------------------------------------","--------","-------","----------") -ForegroundColor DarkGray
    $i = 1
    foreach ($ou in $script:OUData) {
        $name = if ($ou.Name.Length -gt 36) { $ou.Name.Substring(0,36) } else { $ou.Name }
        $restr = "$($ou.HasRestriction)/$($ou.Standards + $ou.Admins)"
        $color = if ($ou.HasRestriction -gt 0) { "Yellow" } else { "White" }
        Write-Host ("  {0,-5} {1,-38} {2,-8} {3,-7} {4,-10}" -f "[$i]",$name,$ou.Standards,$ou.Admins,$restr) -ForegroundColor $color
        $i++
    }
    Write-Host ""
}

function Show-UsersInOU {
    param([PSCustomObject]$OU)
    Write-Host ("  {0,-5} {1,-28} {2,-22} {3,-12}" -f "#","Nom","Login","Restriction") -ForegroundColor Gray
    Write-Host ("  {0,-5} {1,-28} {2,-22} {3,-12}" -f "-----","----------------------------","----------------------","------------") -ForegroundColor DarkGray
    $i = 1
    foreach ($u in $OU.UserList) {
        $r = Get-RestrictionLabel -B $u.logonHours
        $color = switch ($r) { "Active" { "Yellow" }; "Bloque 24h/24" { "Red" }; default { "Green" } }
        $name = if ($u.Name.Length -gt 26) { $u.Name.Substring(0,26) } else { $u.Name }
        Write-Host ("  {0,-5} {1,-28} {2,-22} {3,-12}" -f "[$i]",$name,$u.SamAccountName,$r) -ForegroundColor $color
        $i++
    }
    Write-Host ""
}

# ============================================================
# BOUCLE PRINCIPALE
# ============================================================

$Screen = "MAIN"

while ($true) {

    switch ($Screen) {

        "MAIN" {
            Clear-Host
            Write-Banner "GESTION DES HORAIRES DE CONNEXION AD"
            Show-EnvInfo
            Write-Option "1" "Appliquer des restrictions d'horaires"
            Write-Option "2" "Retirer les restrictions (remettre 24h/24)"
            Write-Option "3" "Consulter les OUs et utilisateurs"
            Write-Option "Q" "Quitter"
            Write-Host ""
            $c = Read-Key -Valid @("1","2","3","q","Q")
            switch ($c) {
                "1" { $script:Action = "RESTRICT"; $Screen = "TARGET" }
                "2" { $script:Action = "REMOVE"; $Screen = "TARGET" }
                "3" { $Screen = "BROWSE_OU" }
                { $_ -in @("q","Q") } { Clear-Host; Write-Host "`n  Au revoir !`n" -ForegroundColor Cyan; exit }
            }
        }

        "BROWSE_OU" {
            Clear-Host
            Write-Banner "CONSULTATION DES OUs"
            Show-OUTable
            Write-Option "0" "Retour"
            Write-Host ""
            $c = Read-Num "  Numero de l'OU (ou 0=retour) : " 1 $script:OUData.Count -AllowZero
            if ($c -eq 0) { $Screen = "MAIN"; continue }
            $script:BrowseOU = $script:OUData[$c - 1]
            $Screen = "BROWSE_DETAIL"
        }

        "BROWSE_DETAIL" {
            Clear-Host
            Write-Banner "OU : $($script:BrowseOU.Name)"
            Write-Info "Standards" "$($script:BrowseOU.Standards)"
            Write-Info "Admins" "$($script:BrowseOU.Admins)"
            Write-Info "Desactives" "$($script:BrowseOU.Disabled)"
            Write-Info "Avec restriction" "$($script:BrowseOU.HasRestriction)"
            Write-Host ""
            Show-UsersInOU -OU $script:BrowseOU
            Write-Option "0" "Retour"
            Write-Host ""
            Read-Key "  Appuyez sur 0 pour revenir : " -Valid @("0")
            $Screen = "BROWSE_OU"
        }

        "TARGET" {
            Clear-Host
            $label = if ($script:Action -eq "RESTRICT") { "APPLIQUER DES RESTRICTIONS" } else { "RETIRER LES RESTRICTIONS" }
            Write-Banner $label
            Write-Host "  Qui cibler ?`n" -ForegroundColor Gray
            Write-Option "1" "Tous les utilisateurs standards ($TotalStd)"
            Write-Option "2" "Selectionner des OUs specifiques"
            Write-Option "3" "Tous les utilisateurs, admins inclus ($($TotalStd + $TotalAdm))"
            Write-Option "0" "Retour"
            Write-Host ""
            $c = Read-Key -Valid @("0","1","2","3")
            if ($c -eq "0") { $Screen = "MAIN"; continue }
            switch ($c) {
                "1" {
                    $script:SelectedUsers = @()
                    foreach ($ou in $script:OUData) { $script:SelectedUsers += $ou.StandardList }
                    $script:TargetDesc = "Tous les standards ($($script:SelectedUsers.Count))"
                    if ($script:Action -eq "RESTRICT") { $Screen = "HOURS" } else { $Screen = "RECAP" }
                }
                "2" { $Screen = "SELECT_OU" }
                "3" {
                    $script:SelectedUsers = @()
                    foreach ($ou in $script:OUData) { $script:SelectedUsers += $ou.UserList }
                    $script:TargetDesc = "Tous ($($script:SelectedUsers.Count), admins inclus)"
                    if ($script:Action -eq "RESTRICT") { $Screen = "HOURS" } else { $Screen = "RECAP" }
                }
            }
        }

        "SELECT_OU" {
            Clear-Host
            $label = if ($script:Action -eq "RESTRICT") { "SELECTIONNER LES OUs A RESTREINDRE" } else { "SELECTIONNER LES OUs (RETRAIT)" }
            Write-Banner $label
            Show-OUTable
            Write-Host "  Entrez les numeros (ex: 1,3,5 ou 1-5 ou tout)" -ForegroundColor Gray
            Write-Option "0" "Retour"
            Write-Host ""
            $sel = Read-MultiSelect "  OUs : " $script:OUData.Count
            if ($sel.Count -eq 1 -and $sel[0] -eq 0) { $Screen = "TARGET"; continue }

            $script:SelectedUsers = @()
            $script:SelectedOUNames = @()
            $hasAdmins = $false

            foreach ($n in $sel) {
                $ou = $script:OUData[$n - 1]
                $script:SelectedOUNames += $ou.Name
                $script:SelectedUsers += $ou.StandardList
                if ($ou.Admins -gt 0) { $hasAdmins = $true }
            }

            if ($hasAdmins) {
                Write-Host ""
                $admCount = ($sel | ForEach-Object { $script:OUData[$_ - 1].Admins } | Measure-Object -Sum).Sum
                Write-Host "  $admCount compte(s) admin detecte(s) dans la selection." -ForegroundColor Yellow
                $incAdm = Read-Key "  Inclure les admins ? (o/n) : " -Valid @("o","n","O","N")
                if ($incAdm -in @("o","O")) {
                    foreach ($n in $sel) { $script:SelectedUsers += $script:OUData[$n - 1].AdminList }
                }
            }

            $script:SelectedUsers = $script:SelectedUsers | Sort-Object SamAccountName -Unique
            $script:TargetDesc = "$($script:SelectedUsers.Count) users dans : $($script:SelectedOUNames -join ', ')"
            $Screen = "CONFIRM_SELECTION"
        }

        "CONFIRM_SELECTION" {
            Clear-Host
            Write-Banner "UTILISATEURS SELECTIONNES ($($script:SelectedUsers.Count))"

            Write-Host ("  {0,-5} {1,-28} {2,-22} {3,-20} {4,-12}" -f "#","Nom","Login","OU","Restriction") -ForegroundColor Gray
            Write-Host ("  {0,-5} {1,-28} {2,-22} {3,-20} {4,-12}" -f "-----","----------------------------","----------------------","--------------------","------------") -ForegroundColor DarkGray

            $i = 1
            foreach ($u in $script:SelectedUsers) {
                $uou = ($u.DistinguishedName -split ',',2)[1]
                $uon = if ($uou -match '^OU=([^,]+)') { $Matches[1] } else { "N/A" }
                $uon = if ($uon.Length -gt 18) { $uon.Substring(0,18) } else { $uon }
                $r = Get-RestrictionLabel -B $u.logonHours
                $color = switch ($r) { "Active" { "Yellow" }; "Bloque 24h/24" { "Red" }; default { "White" } }
                $name = if ($u.Name.Length -gt 26) { $u.Name.Substring(0,26) } else { $u.Name }
                Write-Host ("  {0,-5} {1,-28} {2,-22} {3,-20} {4,-12}" -f "[$i]",$name,$u.SamAccountName,$uon,$r) -ForegroundColor $color
                $i++
            }

            Write-Host ""
            Write-Option "C" "Continuer avec cette selection"
            Write-Option "0" "Retour (changer la selection)"
            Write-Host ""
            $c = Read-Key -Valid @("c","C","0")
            if ($c -eq "0") { $Screen = "SELECT_OU"; continue }
            if ($script:Action -eq "RESTRICT") { $Screen = "HOURS" } else { $Screen = "RECAP" }
        }

        "HOURS" {
            Clear-Host
            Write-Banner "CONFIGURATION DES HORAIRES"
            Write-Host "  Definissez la plage horaire autorisee.`n" -ForegroundColor Gray

            $script:StartHour = Read-Num "  Heure de debut (0-23) : " 0 23
            $script:EndHour = Read-Num "  Heure de fin   (1-24) : " 1 24

            if ($script:StartHour -ge $script:EndHour) {
                Write-Host "  ERREUR : l'heure de debut doit etre avant la fin." -ForegroundColor Red
                Start-Sleep -Seconds 2
                continue
            }

            Write-Host "`n  Jours autorises :`n" -ForegroundColor Gray
            Write-Option "1" "Lundi - Vendredi (jours ouvres)"
            Write-Option "2" "Tous les jours (lundi - dimanche)"
            Write-Option "3" "Personnalise (choisir jour par jour)"
            Write-Option "0" "Retour"
            Write-Host ""
            $dc2 = Read-Key -Valid @("0","1","2","3")
            if ($dc2 -eq "0") { $Screen = "TARGET"; continue }

            [bool[]]$script:DaysEnabled = @($false,$false,$false,$false,$false,$false,$false)
            switch ($dc2) {
                "1" { $script:DaysEnabled = @($false,$true,$true,$true,$true,$true,$false) }
                "2" { $script:DaysEnabled = @($true,$true,$true,$true,$true,$true,$true) }
                "3" {
                    $dn = @("Dimanche","Lundi","Mardi","Mercredi","Jeudi","Vendredi","Samedi")
                    Write-Host ""
                    for ($d = 0; $d -lt 7; $d++) {
                        $rr = Read-Key "  $($dn[$d]) ? (o/n) : " -Valid @("o","n","O","N")
                        $script:DaysEnabled[$d] = ($rr -in @("o","O"))
                    }
                }
            }

            $dnS = @("Dim","Lun","Mar","Mer","Jeu","Ven","Sam")
            $script:ActiveDays = @(); for ($d=0; $d -lt 7; $d++) { if ($script:DaysEnabled[$d]) { $script:ActiveDays += $dnS[$d] } }
            $Screen = "RECAP"
        }

        "RECAP" {
            Clear-Host
            Write-Banner "RECAPITULATIF"
            Write-Info "Domaine" $Domain.DNSRoot
            Write-Info "DC" $DC
            Write-Info "Fuseau" "$($TZ.Id) (UTC${UTCSgn}${UTCOff}) — correction auto"
            Write-Host ""
            if ($script:Action -eq "RESTRICT") {
                Write-Info "Action" "Appliquer restriction"
                Write-Info "Horaires" "$($script:StartHour)h00 - $($script:EndHour)h00"
                Write-Info "Jours" ($script:ActiveDays -join ", ")
            } else {
                Write-Info "Action" "Retirer restrictions (24h/24, 7j/7)"
            }
            Write-Info "Cible" $script:TargetDesc
            Write-Info "Utilisateurs" "$($script:SelectedUsers.Count)"
            Write-Host "`n  -----------------------------------------------`n" -ForegroundColor DarkGray
            Write-Option "O" "Appliquer"
            Write-Option "W" "Simulation (WhatIf)"
            Write-Option "0" "Retour"
            Write-Host ""
            $c = Read-Key -Valid @("o","O","w","W","0")
            if ($c -eq "0") {
                if ($script:Action -eq "RESTRICT") { $Screen = "HOURS" } else { $Screen = "TARGET" }
                continue
            }
            $script:IsWhatIf = ($c -in @("w","W"))
            $Screen = "APPLY"
        }

        "APPLY" {
            Clear-Host
            $ML = if ($script:IsWhatIf) { "SIMULATION" } else { "APPLICATION" }
            Write-Banner "$ML EN COURS..." "Yellow"

            if ($script:Action -eq "RESTRICT") {
                [byte[]]$LH = Build-LogonHours -Start $script:StartHour -End $script:EndHour -DaysEnabled $script:DaysEnabled
                $ActDesc = "Restriction $($script:StartHour)h-$($script:EndHour)h ($($script:ActiveDays -join ', '))"
            } else {
                [byte[]]$LH = @(255,255,255,255,255,255,255,255,255,255,255,255,255,255,255,255,255,255,255,255,255)
                $ActDesc = "Retrait restrictions (24h/24)"
            }

            Write-Log "================================================================"
            Write-Log "$ML - $ActDesc"
            Write-Log "Cible: $($script:TargetDesc) ($($script:SelectedUsers.Count) users)"
            Write-Log "================================================================"

            $SW = [System.Diagnostics.Stopwatch]::StartNew()
            $Tot = $script:SelectedUsers.Count; $App = 0; $Err = 0; $LC = 0
            [array]$Rpt = @()

            foreach ($u in $script:SelectedUsers) {
                $LC++; $Pct = [math]::Round(($LC / $Tot) * 100, 0)
                Write-Progress -Activity $ML -Status "$LC/$Tot ($Pct%)" -PercentComplete $Pct -CurrentOperation $u.Name

                $Bef = Get-RestrictionLabel -B $u.logonHours
                $BefRaw = if ($null -ne $u.logonHours) { ($u.logonHours | ForEach-Object { $_.ToString() }) -join "," } else { "null" }
                $UOU = ($u.DistinguishedName -split ',', 2)[1]
                $UON = if ($UOU -match '^OU=([^,]+)') { $Matches[1] } else { "N/A" }

                try {
                    if ($script:IsWhatIf) {
                        $St = "WHATIF"; Write-Log "WHATIF : $($u.Name) [$UON]" "WARN"
                    } else {
                        Set-ADUser -Identity $u -Replace @{logonHours = $LH} -Server $DC -ErrorAction Stop
                        $St = "OK"; Write-Log "OK : $($u.Name) [$UON]" "OK"
                    }
                    $App++
                    if (-not $script:IsWhatIf) {
                        $Up = Get-ADUser -Identity $u.SamAccountName -Properties logonHours -Server $DC
                        $Aft = Get-RestrictionLabel -B $Up.logonHours
                        $AftRaw = ($Up.logonHours | ForEach-Object { $_.ToString() }) -join ","
                    } else { $Aft = "(simulation)"; $AftRaw = "(simulation)" }
                    $EM = ""
                } catch {
                    try {
                        if (-not $script:IsWhatIf) {
                            Set-ADUser -Identity $u.SamAccountName -Replace @{logonHours = $LH} -Server $DC -ErrorAction Stop
                            $St = "OK (2e)"; $App++
                            $Up = Get-ADUser -Identity $u.SamAccountName -Properties logonHours -Server $DC
                            $Aft = Get-RestrictionLabel -B $Up.logonHours
                            $AftRaw = ($Up.logonHours | ForEach-Object { $_.ToString() }) -join ","
                            $EM = ""; Write-Log "OK (2e) : $($u.Name)" "WARN"
                        }
                    } catch {
                        $St = "ERREUR"; $Aft = "N/A"; $AftRaw = "N/A"
                        $EM = $_.Exception.Message
                        Write-Log "ERREUR : $($u.Name) > $EM" "ERROR"; $Err++
                    }
                }

                $Rpt += [PSCustomObject]@{
                    Status = $St; OU = $UON; Name = $u.Name; SamAccountName = $u.SamAccountName
                    Enabled = $u.Enabled; Restriction_Before = $Bef; Restriction_After = $Aft
                    LogonHours_Before = $BefRaw; LogonHours_After = $AftRaw
                    WhenProcessed = (Get-Date -Format "yyyy-MM-dd HH:mm:ss"); Error = $EM
                }
            }

            Write-Progress -Completed -Activity "Termine"
            $Rpt | Export-Csv -Path $ReportCSV -NoTypeInformation -Encoding UTF8
            $SW.Stop()
            Write-Log "RESUME : $App/$Tot, $Err erreurs, $($SW.Elapsed.ToString('hh\:mm\:ss'))"
            $Screen = "RESULT"
        }

        "RESULT" {
            Clear-Host
            $ML = if ($script:IsWhatIf) { "SIMULATION TERMINEE" } else { "APPLICATION TERMINEE" }
            Write-Banner $ML "Green"
            Write-Info "Utilisateurs traites" "$Tot"
            Write-Info "Appliques" "$App"
            $errColor = if ($Err -gt 0) { "Red" } else { "Green" }
            Write-Host "  Erreurs : " -ForegroundColor Gray -NoNewline
            Write-Host "$Err" -ForegroundColor $errColor
            Write-Info "Duree" "$($SW.Elapsed.ToString('hh\:mm\:ss'))"
            Write-Host ""
            Write-Info "Rapport CSV" $ReportCSV
            Write-Info "Log" $LogFile
            Write-Host ""
            Write-Option "R" "Ouvrir le rapport CSV"
            Write-Option "M" "Retour au menu principal"
            Write-Option "Q" "Quitter"
            Write-Host ""
            $c = Read-Key -Valid @("r","R","m","M","q","Q")
            switch ($c) {
                { $_ -in @("r","R") } { if (Test-Path $ReportCSV) { Invoke-Item $ReportCSV }; continue }
                { $_ -in @("m","M") } {
                    Write-Host "`n  Re-scan en cours..." -ForegroundColor Yellow
                    $script:OUData = @()
                    foreach ($ou in $AllOUs) {
                        $allU = Get-ADUser -Filter * -SearchBase $ou.DistinguishedName -SearchScope OneLevel -Properties Enabled, AdminCount, logonHours
                        $enabled = @($allU | Where-Object { $_.Enabled -eq $true })
                        $disabled = @($allU | Where-Object { $_.Enabled -eq $false })
                        $admins = @($enabled | Where-Object { $_.AdminCount -eq 1 -or $_.SamAccountName -in @("Administrator","Administrateur","krbtgt") })
                        $standards = @($enabled | Where-Object { $_.AdminCount -ne 1 -and $_.SamAccountName -notin @("Administrator","Administrateur","krbtgt") })
                        if ($allU.Count -gt 0) {
                            $restricted = @($enabled | Where-Object { $null -ne $_.logonHours -and ($_.logonHours | Where-Object {$_ -ne 255}).Count -gt 0 }).Count
                            $script:OUData += [PSCustomObject]@{
                                Name=$ou.Name; DN=$ou.DistinguishedName
                                Standards=$standards.Count; Admins=$admins.Count; Disabled=$disabled.Count
                                UserList=$enabled; StandardList=$standards; AdminList=$admins
                                HasRestriction=$restricted
                            }
                        }
                    }
                    $TotalStd = ($script:OUData | Measure-Object -Property Standards -Sum).Sum
                    $TotalAdm = ($script:OUData | Measure-Object -Property Admins -Sum).Sum
                    $TotalDis = ($script:OUData | Measure-Object -Property Disabled -Sum).Sum
                    $Screen = "MAIN"
                }
                { $_ -in @("q","Q") } { Clear-Host; Write-Host "`n  Au revoir !`n" -ForegroundColor Cyan; exit }
            }
        }
    }
}
```
