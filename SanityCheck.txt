<#
    The purpose of this script is to do SQL Server sanity check after SQL Server restart.
#>


function Import-SqlModule {
 

 
    [CmdletBinding()]
    param ()
 
    if (-not(Get-Module -Name SQLPS) -and (-not(Get-PSSnapin -Name SqlServerCmdletSnapin100, SqlServerProviderSnapin100 -ErrorAction SilentlyContinue))) {
    Write-Verbose -Message 'SQLPS PowerShell module or snapin not currently loaded'
 
        if (Get-Module -Name SQLPS -ListAvailable) {
        Write-Verbose -Message 'SQLPS PowerShell module found'
 
            Push-Location
            Write-Verbose -Message "Storing the current location: '$((Get-Location).Path)'"
 
            if ((Get-ExecutionPolicy) -ne 'Restricted') {
                Import-Module -Name SQLPS -DisableNameChecking -Verbose:$false
                Write-Verbose -Message 'SQLPS PowerShell module successfully imported'
            }
            else{
                Write-Warning -Message 'The SQLPS PowerShell module cannot be loaded with an execution policy of restricted'
            }
            
            Pop-Location
            Write-Verbose -Message "Changing current location to previously stored location: '$((Get-Location).Path)'"
        }
        elseif (Get-PSSnapin -Name SqlServerCmdletSnapin100, SqlServerProviderSnapin100 -Registered -ErrorAction SilentlyContinue) {
        Write-Verbose -Message 'SQL PowerShell snapin found'
 
            Add-PSSnapin -Name SqlServerCmdletSnapin100, SqlServerProviderSnapin100
            Write-Verbose -Message 'SQL PowerShell snapin successfully added'
 
            [System.Reflection.Assembly]::LoadWithPartialName('Microsoft.SqlServer.Smo') | Out-Null
            Write-Verbose -Message 'SQL Server Management Objects .NET assembly successfully loaded'
        }
        else {
            Write-Warning -Message 'SQLPS PowerShell module or snapin not found'
        }
    }
    else {
        Write-Verbose -Message 'SQL PowerShell module or snapin already loaded'
    }
 
}

Import-SqlModule

$e = "continue"
$ErrorActionPreference = "stop"; 

$user = "musicandra@gmail.com"  
#$user = "musicandra@gmail.com"   
 
$reportPath = "C:\Test\";  

$reportName = "SQLServiceStatusRpt_$(get-date -format ddMMyyyy).html"; 
 
# Path and Report name together 
$serviceReport = $reportPath + $reportName 
 
$datetime = Get-Date -Format "MM-dd-yyyy_HHmmss"; 
 
# Remove the report if it has already been run today so it does not append to the existing report 
If (Test-Path $serviceReport) { 
    Remove-Item $serviceReport 
} 
 
# Cleanup old files.. 
$Daysback = "-2" 
$CurrentDate = Get-Date; 
$DateToDelete = $CurrentDate.AddDays($Daysback); 
Get-ChildItem $reportPath | Where-Object {$_.name -like "*.html"} | Remove-Item;

$server = (Get-WmiObject Win32_ComputerSystem).Name 

$reboot = Get-WmiObject win32_operatingsystem | Select-Object @{LABEL = 'LastBootUpTime'; EXPRESSION = {$_.ConverttoDateTime($_.lastbootuptime)}} 
#$report += $object
$rebootdatetime = $reboot.LastBootUpTime
$currentdatetime = Get-Date
 
$header = " 
  <html> 
  <head> 
  <meta http-equiv='Content-Type' content='text/html; charset=iso-8859-1'> 
  <title>SQLServer Service Sanity Check</title> 
  <STYLE TYPE='text/css'> 
  <!-- 
  td { 
   font-family: Arial; 
   font-size: 10px; 
   border-top: 1px solid #999999; 
   border-right: 1px solid #999999; 
   border-bottom: 1px solid #999999; 
   border-left: 1px solid #999999; 
   padding-top: 0px; 
   padding-right: 0px; 
   padding-bottom: 0px; 
   padding-left: 0px; 
  }
  h4 {
  font-family: Arial;
  font-weight: lighter;
  font-size: 14px;
  text-align: center;
  color: #003399;
  margin: 0
  } 
  body { 
   margin-left: 5px; 
   margin-top: 5px; 
   margin-right: 0px; 
   margin-bottom: 10px; 
   table { 
   border: thin solid #000000; 
  } 
  --> 
  </style> 
  </head>
  <H2 face='Arial' color='#003399' align='center'>SQL Server Sanity Check after server reboot</H2>
  <H4 face='Arial' color='#003399' align='center'>Server reboot datetime: $rebootdatetime</H4>
  <H4 face='Arial' color='#003399' align='center'>Sanity Check  datetime: $currentdatetime</H4>
  
  <body>   
" 
Add-Content $serviceReport $header
      
# Create and write Table header for report            
$serverheader = "<br /><table width='100%'> 
              <tr bgcolor='#CCCCCC'> 
              <td colspan='4' height='25' align='center'> 
              <font face='Arial' color='#003399' size='4'><strong>$server</strong></font> 
              </td> 
              </tr> 
              </table>"

Add-Content $serviceReport $serverheader            

$serviceheader = "<table width='100%'> 
              <tr bgcolor='#CCCCCC'> 
              <td colspan='4' height='25' align='center'> 
              <font face='Arial' color='#003399' size='3'><strong>SQL Server Services Status</strong></font> 
              </td> 
              </tr> 
              </table>"

Add-Content $serviceReport $serviceheader
             
$tableHeader = " 
             <table width='100%'> 
             <tr bgcolor=#CCCCCC>              
             <td width='5%' align='center'><font size='2'><strong>ServiceName</strong></font></td> 
             <td width='15%' align='center'><font size='2'><strong>ServiceMode</strong></font></td> 
             <td width='10%' align='center'><font size='2'><strong>ServiceState</strong></font></td>
             <td width='10%' align='center'><font size='2'><strong>ServiceMessage</strong></font></td>
             </tr> 
            " 
Add-Content $serviceReport $tableHeader
             
$srvc = Get-WmiObject -query "SELECT * FROM win32_service WHERE name LIKE '%SQL%' AND NOT name LIKE 'MSSQL$%##WID'" -computername $server | Sort-Object -property name;

foreach ($service in $srvc) {            
                                
    $sname = $service.Name
    $smode = $service.startmode
    $sstate = $service.state
    $sstatus = if ($service.state -ne "Running") {"Alarm: Stopped"} else {"OK"}    
            
    if ($sstate -ne "Running") {
        $dataRow = " 
                        <tr> 
                        <td width='10%'align='center'><font color='red'>$sname</font></td> 
                        <td width='5%' align='center'><font color='red'>$smode</font></td> 
                        <td width='15%' align='center'><font color='red'>$sstate</font></td> 
                        <td width='10%' align='center'><font color='red'>$sstatus</font></td>
                        </tr> 
                        " 
        Add-Content $serviceReport $dataRow;                     
    }

    else {
        $dataRow = " 
                        <tr> 
                        <td width='10%'align='center'>$sname</td> 
                        <td width='5%' align='center'>$smode</td> 
                        <td width='15%' align='center'>$sstate</td> 
                        <td width='10%' align='center'>$sstatus</td>
                        </tr> 
                        " 
        Add-Content $serviceReport $dataRow;                     
    }             

}
            
$tableend = "</table>"
Add-Content $serviceReport $tableend
              
$insts = Get-WmiObject -query "SELECT * FROM win32_service WHERE (name LIKE 'MSSQL$%' OR name = 'MSSQLSERVER') AND NOT name LIKE 'MSSQL$%##WID'" -computername $server | Sort-Object -property name;
            
$DBheader = "<table width='100%'> 
              <tr bgcolor='#CCCCCC'> 
              <td colspan='7' height='25' align='center'> 
              <font face='Arial' color='#003399' size='3'><strong>SQL Server Database Status</strong></font> 
              </td> 
              </tr> 
              </table>"

Add-Content $serviceReport $DBheader

$DBStatusheader = " 
             <table width='100%'> 
             <tr bgcolor=#CCCCCC>              
             <td width='5%' align='center'><font size='2'><strong>DateTime</strong></font></td> 
             <td width='15%' align='center'><font size='2'><strong>SQLServerInstanceName</strong></font></td>
             <td width='15%' align='center'><font size='2'><strong>SQLServerEdition</strong></font></td>  
             <td width='10%' align='center'><font size='2'><strong>DBName</strong></font></td>
             <td width='10%' align='center'><font size='2'><strong>DBStatus</strong></font></td> 
             <td width='10%' align='center'><font size='2'><strong>DBUserAccessStatus</strong></font></td> 
             <td width='10%' align='center'><font size='2'><strong>Status</strong></font></td>
             </tr> 
            " 
Add-Content $serviceReport $DBStatusheader
 
foreach ($inst in $insts) { 
                        
    if ($inst.Name -eq "MSSQLSERVER") {$sqlinst = $server} else {$sqlinst = $inst.Name -replace "MSSQL\$" , "$server\"};
                     
    #T-SQL code checks the status of DBs, if all is ONLINE; then it returns OK; #if not it lists the offline DBs
 
    $q = "declare @offline table
                    (instname varchar(50),
					edition sql_variant,
                    dbname sysname,
					dbstatedesc varchar(50),
					dbuseraccessdesc varchar(50),
                    status varchar(50))
                    
                    declare @query varchar(max)
                    declare @all int
                    select @all = count(name) from sys.databases
                    --select @all
                    declare @online int
                    select @online = count(name) from sys.databases where state=0 and user_access_desc = 'MULTI_USER'
                    --select @online
                    if ( @online = @all)
                    begin
                    
                    set @query='select @@servername as instname, SERVERPROPERTY(''Edition'') AS Edition, ''all DBs'' as dbname, ''ONLINE'' as dbstatedesc ,''MULTI_USER'' as dbuseraccessdesc,''OK'' as status '
                    insert into @offline
                    exec (@query)                    
                    
                    select getdate() as datetime, * from @offline
                    end
                    else 
                    begin                    
                    
                    SET @query = 'SELECT @@SERVERNAME AS instname, SERVERPROPERTY(''Edition'') AS Edition, name AS db_name, state_desc, user_access_desc, 
                     CASE 
                        WHEN state_desc = ''OFFLINE'' AND (state <> 0 OR user_access_desc != ''MULTI_USER'') THEN ''Monitoring OFF''
                        WHEN state_desc = ''RECOVERY_PENDING'' THEN ''NOT OK''
                        ELSE ''OK''
                     END AS status 
              FROM sys.databases'

                    
                    insert into @offline
                    exec (@query)                    
                    
                    select getdate() as datetime, * from @offline
                    end"  

                    
    try {
        #Invoke-SQLcmd cmdlet is used to run a query on a SQL instance within #Powershell.
        $DB = @()
        $DB = @(Invoke-Sqlcmd -ServerInstance $sqlinst -Database "master" -Query $q) 
        
        foreach($DBi in $DB)
        {
            $DBdt = $DBi.datetime
            $DBin = $DBi.instname
            $DBe = $DBi.edition
            $DBn = $DBi.dbname
            $DBst = $DBi.dbstatedesc
            $DBu = $DBi.dbuseraccessdesc
            $DBs = $DBi.status
            
            IF ($DBs -ne 'NOT OK' -or $DBst -eq 'RECOVERY_PENDING') {
             $fontColor = 'Green'
    if ($DBst -eq 'RECOVERY_PENDING') {
        $fontColor = 'Red'
    }
                $dataRow = " 
                            <tr> 
                              <td width='10%'align='center'><font color='$fontColor'>$DBdt</font></td> 
                              <td width='5%' align='center'><font color='$fontColor'>$DBin</font></td>
                              <td width='15%' align='center'><font color='$fontColor'>$DBe</font></td> 
                              <td width='10%' align='center'><font color='$fontColor'>$DBn</font></td>
                              <td width='10%' align='center'><font color='$fontColor'>$DBst</font></td>
                              <td width='10%' align='center'><font color='$fontColor'>$DBu</font></td> 
                              <td width='10%' align='center'><font color='$fontColor'>$DBs</font></td>
                            </tr>
                            " 
                Add-Content $serviceReport $dataRow;                         
            }
                    
            else {
                $dataRow = " 
                            <tr> 
                            <td width='10%'align='center'>$DBdt</td> 
                            <td width='5%' align='center'>$DBin</td>
                            <td width='15%' align='center'>$DBe</td> 
                            <td width='10%' align='center'>$DBn</td>
                            <td width='10%' align='center'>$DBst</td>
                            <td width='10%' align='center'>$DBu</td> 
                            <td width='10%' align='center'>$DBs</td>
                            </tr> 
                            " 
                Add-Content $serviceReport $dataRow;                         
            }

        }
    }
    Catch {
        $dataRow = "  
                    <tr> 
                    <td align='center' colspan='7'><font color='red'>SQL Server instance $sqlinst on $server threw an error while checking DB status.</font></td> 
                    </tr>
                    "
        Add-Content $serviceReport $dataRow;                     
        $ErrorActionPreference = $e
    }

    $ErrorActionPreference = $e;                
                                                
}

$tableend1 = "</table>"
Add-Content $serviceReport $tableend1   

$tableDescription = "<H5 face='Arial' color='#003399' align='left'><b><u>Guidelines:</u></b></H5>
<p face='Arial' font-size=9px>1) Hello team, This is sanity check report after server reboot. It is designed to minimize our efforts that we have to put in for SQL Server healthcheck after server reboot. please note that we only have to login to the server if we see any issues in this report. If everything is ok, we just have to resolve the tickets by Refering the status of this report in ticket comments.</p>
<p face='Arial' font-size=9px>2) Any disabled service does not need to be looked into and should be ignored.</p>"


Add-Content $serviceReport $tableDescription 
Add-Content $serviceReport "</body></html>" 

# Send Notification 
 
<#Write-Host "Sending Email notification to $user" 
   
$smtpServer = "dba.smtp.com" 
$smtp = New-Object Net.Mail.SmtpClient($smtpServer) 
$msg = New-Object Net.Mail.MailMessage 
$msg.To.Add($user) 
$msg.From = "SQLAlerts-$musicandra@gmail.com" 
$msg.Subject = "CompanayName: SQL Server Sanity Check - $server" 
$msg.IsBodyHTML = $true 
$msg.Body = get-content $serviceReport 
$smtp.Send($msg) 
$body = ""#>



$EmailFrom = “pedd@outlook.com”
$EmailTo = “musicanddddra@gmail.com”
$Subject = "Scanity Check Report for " + $date
#$Body = “Disk Space Alerts”
$SMTPServer = “smtp.outlook.com”
$SMTPClient = New-Object Net.Mail.SmtpClient($SmtpServer, 587)
$SMTPClient.EnableSsl = $true
$SMTPClient.Credentials = New-Object System.Net.NetworkCredential(“emailid@outlook.com”, “Giveurpassword”);
$mailMessage = New-Object System.Net.Mail.MailMessage($EmailFrom, $EmailTo, $Subject, $Body)
$attachmentPath = "$serviceReport"
$attachment = New-Object System.Net.Mail.Attachment($attachmentPath)
$mailMessage.Attachments.Add($attachment)
$SMTPClient.Send($mailMessage)
