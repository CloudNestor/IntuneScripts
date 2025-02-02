#============================================================================================
$TenantID = '<your tenantID>'
$ClientID = '14d82eec-204b-4c2f-b7e8-296a70dab67e' # Microsoft Graph PowerShell #

$token = Get-MsalToken -TenantId $TenantID -ClientId $ClientID -Interactive

$AuthToken = @{'Authorization' = $token.CreateAuthorizationHeader()}
#=============================================================================================

$DeviceID = @()
$global:EnrolledDevices = @()
$myobj = @()

Try{
    $DeviceID = Import-Csv -path $ENV:USERPROFILE\Downloads\DevicetoCheck.csv -delimiter ";"
    If($null -eq $DeviceID[0].ReferenceID){
        Write-host "Import file incorrect" -ForegroundColor Red
        exit 1
    }
}catch{
    write-host "Input file not found" -ForegroundColor Red
    exit 1
}

ForEach ($ID in $DeviceID){
    Try{
        $EnrollState = (Invoke-RestMethod -Headers $authToken -Method Get -Uri "https://graph.microsoft.com/beta/admin/windows/updates/updatableAssets/$($ID.ReferenceId)").enrollment.feature.enrollmentstate
    }catch{
        If($error[0] -like "*403*"){
            write-host "Check your permissions" -ForegroundColor Red
            Exit 1
        }elseif($error[0] -like "*404*"){
            $Action = "Not found"
        }
    }

    If($EnrollState -eq "enrolling"){
        # Delete device
        Try{
            Invoke-RestMethod -Headers $authToken -Method Delete -Uri "https://graph.microsoft.com/beta/admin/windows/updates/updatableAssets/$($ID.ReferenceId)"
        }catch{
            If($error[0] -like "*403*"){
                write-host "Check your permissions" -ForegroundColor Red
                Exit 1
            }
        }

        #Re-add device
        $object = New-Object –TypeName PSObject
        $subobject = New-Object –TypeName PSObject
        $subobject | Add-Member -MemberType NoteProperty -Name '@odata.type' -Value “#microsoft.graph.windowsUpdates.azureADDevice”
        $subobject | Add-Member -MemberType NoteProperty -Name 'id' -Value $($ID.ReferenceId)
        $object | Add-Member -MemberType NoteProperty -Name 'updateCategory' -Value “feature”
        $object | Add-Member -MemberType NoteProperty -Name 'assets' -Value @($subobject)
        $JSON = $object | ConvertTo-Json
        Invoke-RestMethod -Headers $authToken -Method Post -Uri "https://graph.microsoft.com/beta/admin/windows/updates/updatableAssets/enrollassets" -Body $JSON -ContentType "application/json"
        $Action = "Re-added"
    }

    $myObj = "" | Select "DeviceID", "EnrollmentState", "Action" 
    $myObj.DeviceID = $ID.ReferenceId
    $myObj.enrollmentState = $EnrollState
    $MyObj.action = $Action
    $global:EnrolledDevices += $myObj
    $myObj = $Null
    $EnrollState = @()
    $Action = @()
}

$global:EnrolledDevices | export-csv "$ENV:USERPROFILE\Downloads\FUCheck.csv" -notype -delimiter ";"
