# Recovery VHDX and ISO files for Crowdstrike kernel driver failure on 19/07/2024

THIS STILL NEEDS MORE TESTING BEFORE IT SEES WIDE PRODUCTION USAGE

You can either "trust me" and use the binaries I have posted at https://github.com/adappt/CrowdStrike-WinPE-Recovery/releases/tag/0.1 or you can "distrust me" (reccomended) and make you own as below:

To make your own: (Note: These instructions are pretty rushed, but they should get the gist of it across, I will improve them!)

Note: To upload to Azure, create a VHDX and convert it using Convert-VHD e.g ```Convert-VHD -Path C:\WinPE_amd64-EFI2011CA-workingautoshutdown.vhdx -DestinationPath C:\WinPE_amd64-EFI2011CA-workingautoshutdown-fixed.vhd -VHDType Fixed```

TEST THIS IN YOUR ENVIRONMENT BEFORE TRYING ON A PROD MACHINE. THIS WAS MADE IN A RUSH.

1 - Download and install the Windows ADK from https://go.microsoft.com/fwlink/?linkid=2271337

2 - Download and install the Windows ADK PE add-on from https://go.microsoft.com/fwlink/?linkid=2271338

3 - Follow instructions in this section: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#update-the-windows-pe-add-on-for-the-windows-adk

When following step 5 (https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-mount-and-customize?view=windows-11#add-updates-to-winpe-if-needed) use this URL (https://catalog.sf.dl.delivery.mp.microsoft.com/filestreamingservice/files/2546b8fa-11f3-4a37-b9ea-7667e1478547/public/windows11.0-kb5040438-x64_58a8df174fda3cd4cedc651e904a579a896d4d59.msu) to download the update file, it's not really that well documented.

When following step 6, you need to choose between the 2011 and 2023 UEFI CA, though most systems with Secure Boot use 2011, so I suggest starting with this.

4 - Mount the image with ```Dism /Mount-Image /ImageFile:"C:\WinPE_amd64\media\sources\boot.wim" /index:1 /MountDir:"C:\WinPE_amd64\mount"```

4 - Whilst the image is mounted, navigate to C:\WinPE_amd64\mount\Windows\System32\ and open Startnet.cmd and add the below content to the bottom:

```
@echo off
setlocal enabledelayedexpansion

:: Define drive letters to check
set "drives=A B C D E F G H I J K L M N O P Q R S T U V W X Y Z"

:: Loop through all drive letters
for %%i in (%drives%) do (
    set "drive=%%i:"
    echo Checking drive %drive%...
    
    :: Navigate to the drivers folder
    pushd "%drive%\Windows\System32\drivers\CrowdStrike\" 2>nul && (
        :: Check for files matching the pattern
        for %%f in (C-00000291*.sys) do (
            echo Found file: %%f
            
            :: Check if file does not end with .bak
            set "originalName=%%~nf"
            if /i not "!originalName!"==".bak" (
                :: Rename the file by adding .bak extension
                set "newName=!originalName!.bak"
                ren "%%f" "!newName!"
                echo File renamed to: !newName!
            ) else (
                echo File already ends with .bak, skipping rename.
            )
        )
        
        popd
    ) || (
        echo Drive %drive% does not exist or access denied.
    )
)

wpeutil shutdown
```

This will rename the affected drivers with a .bak file extension.

5 - Close an open editors/file browsers and execute ```Dism /Unmount-Image /MountDir:"C:\WinPE_amd64\mount" /commit```

6 - Create the VHDX/ISO using instructions in https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-create-usb-bootable-drive?view=windows-11#create-a-winpe-iso-dvd-or-cd
