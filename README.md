# Recovery VHDX, USB and ISO creation to remediate CrowdStrike kernel driver failure on 19/07/2024

Please test this extensively before using in production.

Please build this yourself if you intend to use this in production.

Estimated time spent reading, and following this is 30-60 minutes.

Booting this will delete the problematic driver on any connected drives on the machine.

This has been tested, and should work fine on most systems. You may need to inject drivers for things like RAID cards which are not supported in WinPE by default. Please refer to this link below for such systems.

https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/add-and-remove-drivers-to-an-offline-windows-image?view=windows-11

To make your own VHDX or ISO or USB: 

1 - Download and install the Windows ADK from https://go.microsoft.com/fwlink/?linkid=2271337

2 - Download and install the Windows ADK PE add-on from https://go.microsoft.com/fwlink/?linkid=2271338

3 - Follow steps 2,3,4 in this section: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#update-the-windows-pe-add-on-for-the-windows-adk

4 - Download the culminative update file for Windows PE (2024-07) from https://catalog.sf.dl.delivery.mp.microsoft.com/filestreamingservice/files/2546b8fa-11f3-4a37-b9ea-7667e1478547/public/windows11.0-kb5040438-x64_58a8df174fda3cd4cedc651e904a579a896d4d59.msu

5 - Follow steps 2,3 in this section and apply the update file from the previous step: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-mount-and-customize?view=windows-11#add-updates-to-winpe-if-needed

6 - Follow step 6 in this section. If your affected system(s) deploy UEFI Secure Boot, they are most likely using the UEFI 2011 CA, so follow only those steps, otherwise use UEFI 2023 CA. https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#update-the-windows-pe-add-on-for-the-windows-adk

7 - Open "C:\WinPE_amd64\mount\Windows\System32\startnet.cmd" in Notepad or an editor of your choice. Append the below to this file, do not remove the ```wpeinit``` command at the start of it:

```
@echo off
@cls

echo "The script for removing the CrowdStrike malformed kernel driver has started!"
echo LIST VOL>>VOLLIST.TXT
diskpart /S VOLLIST.TXT>>VOLTEMP.TXT
setlocal enabledelayedexpansion
set "filePath=VOLTEMP.txt"
set "assignlist=ASSIGN.txt"
set "filePointer=%assignlist%"
:: Initialize counter
set /a volumeCount=0
:: Read the file line by line
for /f "tokens=* delims=" %%a in ('type "%filePath%"') do (
    :: Check if the line starts with "Volume"
    if "%%a" neq "" (
        set "line=%%a"
        :: Assuming the line ends with a space or end of line character
        if "!line:~-1!"==" " (
            echo SELECT VOL !volumeCount! >> "%filePointer%"
            echo ASSIGN >> "%filePointer%"
            set /a volumeCount+=1
        )
    )
)
:: Display the total count
echo Number of volumes: !volumeCount!
diskpart /S ASSIGN.TXT
:: Define drive letters to check
set "drives=A B C D E F G H I J K L M N O P Q R S T U V W X Y Z"
:: Loop through all drive letters
for %%i in (!drives!) do (
    set "drive=%%i:"
    echo Checking drive !drive! ...
    :: Navigate to the drivers folder
    pushd "!drive!\Windows\System32\drivers\CrowdStrike\" 2>nul && (
        :: Check for files matching the pattern
        for %%f in (C-00000291*.sys) do (
            echo Found file: %%f
            del %drive%\Windows\System32\drivers\CrowdStrike\%%f
        )
        popd
    ) || (
        echo Drive !drive! does not exist probably
    )
)

wpeutil shutdown
```

8 - Follow step 1 at this section https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-mount-and-customize?view=windows-11#unmount-the-windows-pe-image-and-create-media

9 - Execute ```rmdir C:\WinPE_amd64 /S /Q``` followed by ```copype amd64 C:\WinPE_amd64``` to copy installation files usable to make an ISO/USB/VHDX

9.5 - (optional) You can remove the "Press any key to boot from CD or DVD..." screen by deleting C:\WinPE_amd64\media\Boot\bootfix.bin , this is likely what most users want

10 - (optional) Use the step described here to make an ISO https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#create-a-winpe-iso-dvd-or-cd

11 - (optional) Use the step described here to make a VHDX https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#create-a-windows-pe-vhd-to-use-with-hyper-v

12 - (optional) Use the step described here to make a USB https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#create-a-bootable-windows-pe-usb-drive

Note: To upload to Azure, create a VHDX and convert it using Convert-VHD e.g 

```Convert-VHD -Path C:\nameofyour.vhdx -DestinationPath C:\nameofyournew.vhd -VHDType Fixed```
