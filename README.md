# How to self-sign a Windows package created with Pyinstaller  -

This document aims to explain all the necessary steps to self-sign a Windows executable.

> :warning: **Warning**  
> Some of the commands provided need to be completed. The fields to complete are indicated by the characters `<` and `>`.

## Prerequisites

Please make sure to match all the prerequisite before starting the process of signing the package.

### Using Pyinstaller

To use these instructions, you first need to learn how to package a Python script with Pyinstaller using a spec file. If you do not master this step yet, I invite you to visit the [Pyinstaller documentation](https://pyinstaller.org/en/stable/spec-files.html).

### Install a Windows Source Development Kit (SDK)

To complete the steps, you will need to install a Windows SDK in order to have the programs needed to sign the package. You will find the installer on the offical [Windows SDK page](https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/).

> :warning: **Warning**  
> During the installation, you will be asked which parts of the SDK to install, only the Signing Tools part is necessary here. It allows you to install a few Mb instead of several Gb.

### Add Signing Tools to your environment variables

Once Signing Tools is installed, you need to add it to your path to be able to use it in Powershell. To do this open the folder containing the binaries of the SDK. You should find it in a place like this `/Program Files (x86)/Windows Kits/10/bin/10.0.22621.0/x64`, copy the path and add it to your environment variables.

If you do not know how to edit them, I invite you to visit this [website](https://docs.oracle.com/en/database/oracle/machine-learning/oml4r/1.5.1/oread/creating-and-modifying-environment-variables-on-windows.html#GUID-DD6F9982-60D5-48F6-8270-A27EC53807D0).

### Update your Powershell version

To create a certificate, we need an other tool called `New-SelfSignedCertificate`, unfortunately it is not include in the old versions of Powershell. Try to call the tool in the terminal and if the error indicates that the command is not found, you need to update Powershell to the last version before going further.

## Include version informations

You first need to include version informations in your executable. This is done by specifying a version file in argument inside the Pyinstaller spec file. This will add information into the manifest of your application which is visible when you open the properties of the file in the file explorer.

Here is a full example of a version file:

```
# UTF-8
#
# For more details about fixed file info 'ffi' see:
# http://msdn.microsoft.com/en-us/library/ms646997.aspx
VSVersionInfo(
  ffi=FixedFileInfo(
# filevers and prodvers should be always a tuple with four items: (1, 2, 3, 4)
# Set not needed items to zero 0.
filevers=(1, 0, 0, 0),
prodvers=(1, 0, 0, 0),
# Contains a bitmask that specifies the valid bits 'flags'r
mask=0x3f,
# Contains a bitmask that specifies the Boolean attributes of the file.
flags=0x0,
# The operating system for which this file was designed.
# 0x4 - NT and there is no need to change it.
OS=0x4,
# The general type of file.
# 0x1 - the file is an application.
fileType=0x1,
# The function of the file.
# 0x0 - the function is not defined for this fileType
subtype=0x0,
# Creation date and time stamp.
date=(0, 0)
),
  kids=[
StringFileInfo(
  [
  StringTable(
    u'040904B0',
    [StringStruct(u'CompanyName', u'Your company name'),
    StringStruct(u'FileDescription', u'Your Filename'),
    StringStruct(u'FileVersion', u'Your version number'),
    StringStruct(u'InternalName', u'Your app name'),
    StringStruct(u'LegalCopyright', u'Copyright (c) your company name'),
    StringStruct(u'OriginalFilename', u'YourApp.exe'),
    StringStruct(u'ProductName', u'YourApp'),
    StringStruct(u'ProductVersion', u'4.2.0')])
  ]),
VarFileInfo([VarStruct(u'Translation', [1033, 1200])]) 
  ]
)
```

And here is how you specify the version file in your spec file for Pyinstaller :

```Python
...
exe = EXE(pyz,
          a.scripts,
          a.binaries,
          a.zipfiles,
          a.datas,
          name='YourApp',
          version="version.txt",
          ...)
...
```

Once you have recompiled your executable with these parameters, you can go to the next step.

## Generate the Certificate

You can now really start to manipulate certificates. You can create a new one for your application using the following command:

```Powershell
New-SelfSignedCertificate -Type Custom -Subject "CN=<Your company name>" -KeyUsage DigitalSignature -FriendlyName "YourApp" -CertStoreLocation "Cert:\CurrentUser\My" -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.3", "2.5.29.19={text}")
```

To see if the command succeeded, enter the following commands and search for the `FriendlyName` you just gave to your certificate:

```Powershell
Set-Location Cert:\CurrentUser\My
Get-ChildItem | Format-Table FriendlyName, Thumbprint, Subject
```

Save the thumbprint displayed next to the name of your certificate, you will need it for the next step.

## Export the certificate

Before going further I invite you to restart your terminal to make sure it takes into account the new certificate you just created.

The next step is now to export the certificate in a `.pfx` file to sign your executable with it. This file will be protected with a password and contain your certificate. To do this, use the following commands and make sure to replace the password with the one you choose and the thumbprint with the one from the previous step:

```Powershell
$password = ConvertTo-SecureString -String "<Your Password>" -Force -AsPlainText 
Export-PfxCertificate -cert "Cert:\CurrentUser\My\<YourThumbprint>" -FilePath certificate.pfx -Password $password
```

It will create a `certificate.pfx` in your current directory, move it close to your executable to ease the next step.

## Sign the package

It is now finally time to sign the executable with the certificate. To do this use the following command:

```Powershell
signtool sign /f ./pyinstaller_config/cert.pfx /p <password> /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 <YourApp.exe>
```

If you need to sign an other version of the same package, you can use the same certificate and simply use this last command again.
