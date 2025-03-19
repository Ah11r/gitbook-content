# Windows Forensics 1

To gain familiarity with the Zimmerman's tool, I covered the Windows Forensics 1 room on TryHackMe. Below are my key takeaways which will be a reference in the upcoming tasks.

#### Registry(regedit.exe)

A collection of databases that contain the system's configuration data include.

```
1. HKEY_CLASSES_ROOT
2. HKEY_CURRENT_USER - contains the root of config information for the user currently logged on
3. HKEY_LOCAL_MACHINE - contains config info particular to the computer, for any user
4. HKEY_USERS - contains all the actively loaded user profiles on the computer
5. HKEY_CURRENT_CONFIG - contains info about hardware profile that is used by the local computer at system startup
```

#### Offline Registry Access

In a scenario where we have access to the offline disk, the following are the locations of registry hives.

```
	1. *DEFAULT - HKEY_USERS\DEFAULT
	2. *SAM - HKEY_LOCAL_MACHINE\SAM
	3. *SECURITY - HKEY_LOCAL_MACHINE\Security
	4. *SOFTWARE - HKEY_LOCAL_MACHINE\Software
	5. *SYSTEM - HKEY_LOCAL_MACHINE\System
```

#### System info and accounts

We can get the OS Version in the following location on the registry:

```
	 SOFTWARE\Microsoft\Windows NT\CurrentVersion
```

Current Control set:- hives containing the machine's configuration data used for controlling system startup.\
ControlSet001 - points to the control set that a machine booted with.\
These can be found at:

```
HKLM\SYSTEM\CurrentControlSet - created when the machine is live (most accurate info)
HKLM\SYSTEM\Select\Current
HKLMSYSTEM\Select\LastKnownGood
```

Computer Name:- helps verify that we are working on the correct machine. It can be obtained at:

```
SYSTEM\CurrentControlSet\Control\ComputerName\ComputerName
```

Time Zone Information:- helps understand the chronology of events as they happened

```
	SYSTEM\CurrentControlSet\Control\TimeZoneInformation
```

To obtain the Network Interfaces and Past Networks:

```
SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces

SOFTWARE\Microsoft\WindowsNT\CurrentVersion\NetworkList\Signatures\Unmanaged

SOFTWARE\Microsoft\WindowsNT\CurrentVersion\NetworkList\Signatures\Managed
	
```

Autostart Programs (Autoruns):- contains info about programs or commands that run when a user logs on. In the registry, they can be found at:

```
NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Run

NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\RunOnce

SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce

SOFTWARE\Microsoft\Windows\CurrentVersion\policies\Explorer\Run

SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

Autostart services:

```
SYSTEM\CurrentControlSet\Services
```

SAM hive and user information:- contain user account info, login info, group info

```
LOCATION:
	SAM\Domains\Account\Users
```

#### File/folder usage or knowledge

Recent Files:- Contains a list of recently opened files for each user

```
LOCATIONS:

NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs
```

Office Recent Files:- a list of recently opened office documents

```
LOCATIONS:

NTUSER.DAT\Software\Microsoft\Office\VERSION(version 15.0 refers to office 2013, 16.0 refers to office 2016,2019,2021)

NTUSER.DAT\Software\Microsoft\Office\VERSION\UserMRU\LiveID_####\FileMRU
```

ShellBags:- contains info about the windows shell which can be used to identify the most recently used files and folders

```
LOCATIONS:

USRCLASS.DAT\Local Settings\Software\Microsoft\Windows\Shell\Bags

USRCLASS.DAT\Local Settings\Software\Microsoft\Windows\Shell\BagMRU

NTUSER.DAT\Software\Microsoft\Windows\Shell\BagMRU

NTUSER.DAT\Software\Microsoft\Windows\Shell\Bags
```

Open/Save and LastVisited Dialog MRUs:-Contains info about the recent open/save locations. This can help in finding out recently used files as well.

```
LOCATIONS:

NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\OpenSavePIDlMRU

NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\ComDlg32\LastVisitedPidlMRU
```

Windows Explorer Address/Search Bars:-Contain info about user's recent explorer path and searches

```
LOCATIONS:

NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\TypedPaths

NTUSER.DAT\Software\Microsoft\Windows\CurrentVersion\Explorer\WordWheelQuery
```

#### External/USB device forensics

To identify external devices plugged into the machine, we can obtain this info at:

```
SYSTEM\CurrentControlSet\Enum\USBSTOR
SYSTEM\CurrentControlSet\Enum\USB
```

For the First/Last Times of connection:

```
SYSTEM\CurrentControlSet\Enum\USBSTOR\Ven_Prod_Version\USBSerial#\Properties\{83da6326-97a6-4088-9453-a19231573b29}\####
	Oo64=first connection
	0066=last connection
	0067=last removal

USB device Volume Name:
	SOFTWARE\Microsoft\Windows Portable Devices\Devices
```

#### Evidence of execution

UserAssist:-Contain info about the programs launched,no of times launched, and time of their launch

```
LOCATION:

NTUSER.DAT\Software\Microsoft\Windows\Currentversion\Explorer\UserAssist\{GUID}\Count
```

ShimCache:(also called AppCompatCache)-contain tracks of all applications launched i.e. file name, size and last modified date

```
LOCATION:

SYSTEM\CurrentControlSet\Control\Session Manager\AppCompatCache
```

AmCache:- Contains additional info to Shimcache. i.e. execution path, installation,execution and deletion times, and SHA1 hashes of the executed programs

```
LOCATION:

C:\Windows\appcompat\Programs\Amcache.hve
	Amcache.hve\Root\File\{Volume GUID}\
```

BAM/DAM:-Background Activity Monitor keeps a tab on the activity of backround applications. Desktop Activity Monitor optimizes the power consumption of the device. They contain info about last run programs, their full programs and last execution time.

```
LOCATION:

SYSTEM\CurrentControlSet\Services\bam\UserSettings\{SID}

SYSTEM\CurrentControlSet\Services\dam\UserSettings\{SID}
```

In Part 2 of this particular article, I will be discussing how I tackled the room questions using the Eric Zimmerman's tools. This part 2 will be under the category [TryHackMe](https://ah11r.github.io/categories/tryhackme/).
