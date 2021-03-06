## Important note

macOS 10.12.4 broke createOSXinstallPkg. Update to the current GitHub code to regain partial functionality: you will not be able to add additional packages, and you'll have to provide your own copy of `brtool` from previous macOS Sierra installer apps or older createOSXinstallPkg-created packages.

Due to Apple changes, it's likely-to-virtually-certain that some functionality of createOSXinstallPkg is gone forever: nothing we do will bring back the ability to insert additional packages to be installed.

I think it's probably better longer-term for the community to focus on new tools and new workflows using the `startosinstall` binary inside the Install macOS.app. Munki 3 supports this workflow.

If you need to install macOS with additional packages, I suggest you continue using createOSXinstallPkg with a 10.12.3 installer, and then allow Apple Software Update to upgrade machines to 10.12.4 and beyond.

## createOSXinstallPkg

This tool allows you to create an installer package from an "Install OS X.app" or an InstallESD.dmg. You can use this package to install OS X on an empty partition, but perhaps more interestingly, you can also use it to upgrade existing OS X installations to a newer version of the OS X. There are many tools and workflows that support the installation of Apple packages; you can use these together with an OS X installation package to upgrade machines to the latest version of OS X.

This is especially interesting when used with tools like [Munki](https://github.com/munki/munki) -- you can automate the upgrade of a group of machines while still preserving user data, or offer an upgrade as a "Self-Service"-type option where a user can initiate an OS X upgrade themselves without needing to have administrative rights.

[_Please read this important 10.10+ note_](#further-note-on-additional-packages-and-yosemite-and-el-capitan)

### Getting Started

#### What you need

This toolset, which includes:

    createOSXinstallPkg
    Resources/installosxpkg_postflight

You may put the toolset anywhere you'd like, but keep the Resources folder and its contents in the same directory as `createOSXinstallPkg`.

You'll also need an installation source for OS X (10.7 through 10.11 currently supported): a copy of the "Install Mac OS X Lion.app", "Install OS X Mountain Lion.app", "Install OS X Mavericks.app", etc, install application you can get from the Mac App Store.

Finally, and most importantly, you'll need the rights to install the OS X version on the machines you manage. Just because this tool allows you to create an OS X installation package does not mean it is legal for your organization to install it on all your Macs.

(Since Mavericks, Yosemite, and El Capitan are free, one assumes you can install them with abandon. However, I am not a lawyer, and this does not constitute advice or a recommendation.)

#### How to use it

You must run `createOSXinstallPkg` with root privileges.

    sudo ./createOSXinstallPkg --source /path/to/Install\ OS\ X\ Mountain\ Lion.app

This creates an installation package in the current directory named `InstallOSX_[version]_[build].pkg`, where "version" and "build" are the version and build numbers of the OS X version that will be installed.

    sudo ./createOSXinstallPkg --source /path/to/Install\ OS\ X\ Mountain\ Lion.app --output /path/to/some/directory/

    sudo ./createOSXinstallPkg --source /path/to/Install\ OS\ X\ Mountain\ Lion.app --output /path/to/output.pkg

Adding the `--output` option allows you to specify an alternate location and/or name for the output package.

    sudo ./createOSXinstallPkg --source /path/to/Install\ Mac\ OS\ X\ Lion.app --pkg /path/to/LocalAdmin.pkg --pkg /path/to/DisableSetupAssistant.pkg

The `--pkg` option allows you to add one or more packages to be installed after the OS is installed. You may specify multiple packages. They will be installed in the order given at the command line.

    sudo ./createOSXinstallPkg --source /path/to/Install\ Mac\ OS\ X\ Lion.app --identifier 'com.example.installosx.pkg'

The `--identifier` option allows you to change the package identifier, which is by default 'com.googlecode.munki.installosx.pkg'.

    sudo ./createOSXinstallPkg --plist /path/to/xml.plist

Options to `createOSXinstallPkg` can be stored in a plist file. This allows you to save the "ingredients" and "recipe" for a package for future reuse.

Example plist:

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Source</key>
        <string>/Volumes/Data/Applications/Install OS X Mountain Lion.app</string>
        <key>Output</key>
        <string>/Volumes/Data/OSX_Install_Packages</string>
        <key>Packages</key>
        <array>
            <string>/Volumes/Data/Packages/LocalAdminAcct.pkg</string>
            <string>/Volumes/Data/Packages/DisableSetupAssistant.pkg</string>
            <string>/Volumes/Data/Packages/munkitools-0.8.3.1610.0.mpkg</string>
            <string>/Volumes/Data/Packages/munki_kickstart.pkg</string>
        </array>
        <key>Identifier</key>
        <string>com.example.installmountainlion.pkg</string>
    </dict>
    </plist>

If an option is specified in the plist and also explicitly at the command-line, the command-line "wins". (Note this means also that since multiple packages can be specified, that packages in a plist and packages at the command-line are _not_ merged; the packages given at the command-line are the _only_ packages used.)


#### How it works

The package generated by `createOSXinstallPkg` is a 'payload-free' package -- that is, it does not install anything from the traditional Archive.pax.gz payload found in most packages. Instead, the real work is done as a package postflight script located at `[PACKAGE]/Contents/Resources/postflight`.

The postflight script performs the actions that the GUI "Install Mac OS X Lion" or "Install OS X Mountain Lion" application does when you choose to install OS X.

Those actions are:

  1. Create an `OS X Install Data` directory at the root of the target volume.
  2. Mount the `InstallESD.dmg` disk image.
  3. Copy the `kernelcache` and `boot.efi` files from the disk image to the `OS X Install Data` directory. (The `kernelcache` is copied to the Recovery HD helper partition if the target volume is encrypted with FileVault 2.)
  4. Unmount (eject) the `InstallESD.dmg` disk image.
  5. If the `InstallLion.pkg` is on the same volume as the target volume, create a hard link to the `InstallESD.dmg` disk image in `OS X Install Data`, otherwise copy the `InstallESD.dmg` disk image to that directory.
  6. Create a `com.apple.Boot.plist` file in the `Mac OS X Install Data` directory which tells the kernel how to mount the disk image to use for booting. (This file is instead created on the the Recovery HD helper partition if the target volume is encrypted with FileVault 2.)
  7. Create a `minstallconfig.xml` file, which tells the OS X Installer what to install and to which volume to install it. It also provides a path to a `MacOSXInstaller.choiceChanges` file if one has been included in the package.
  8. Create an `index.sproduct` file and an `OSInstallAttr.plist` in the `OS X Install Data` directory. These are also used by the OS X Installer.
  9. Set a variable in nvram that the OS X Installer uses to find the product install info after reboot.
  10. Use the `bless` command to cause the Mac to boot from the kernel files copied to the `OS X Install Data` directory.
  11. If the target volume is a Core Storage or Apple RAID volume, setup/update boot helper partitions (these are the ones that show up as type "Apple_Boot" in the output of `diskutil list`).

Since most of the work is done with a postflight script, and since that script may need to do a lengthy copy of around 4GB of data (if the package is not on the target volume), you may see a long delay at the "Running package scripts" stage of installation. This is normal. (Annoyingly, the Installer.app displays "Install time remaining: Less than a minute" for several minutes during this stage.)

The next step would be to reboot, but the postflight script does not do this; it just exits. The package is marked as requiring a reboot, so whatever mechanism is used to install the package is responsible for rebooting as soon as possible after the install.

Upon reboot, the machine boots and runs the OS X Installer just as if you had run the "Install Mac OS X Lion" or "Install OS X El Capitan" application manually. It creates or updates a "Recovery HD" partition and installs OS X on the target volume, displaying the OS X Installer GUI. When installation is complete, the machine reboots a second time, this time booting from the new OS X installation.


#### Preinstall checks

A "Distribution" file, located at `InstallOSX.pkg/Contents/distribution.dist` controls the InstallCheck and VolumeCheck logic.

`createOSXinstallPkg` copies InstallCheck and VolumeCheck logic from the OSInstall.mpkg found in the Install.app or InstallESD.dmg. This means that the resulting package will use the same logic as Apple when deciding if a machine/volume is a valid install destination. (One exception -- `createOSXinstallPkg` disables the check for command-line installs; without disabling this check you would not be able to install Lion or Mountain Lion using Munki or ARD or any other mechanism that uses the command-line `/usr/sbin/installer`.)

The `distribution.dist` declares the install size is 8388608 KB (8* 1024 * 1024 KB, or 8GB), which should prevent attempted installation on volumes with less than 8GB free space. You can edit this number if you'd like.


### Customizing the install

#### Installer choice changes

You'll find a `MacOSXInstaller.choiceChanges` file at `InstallOSX.pkg/Contents/Resources/OS X Install Data/MacOSXInstaller.choiceChanges`

See `man installer` for more info on ChoiceChangesXML files.


#### Additional packages

The most likely customization you will want to do is to add additional packages to be installed after the OS install. Some examples might include a package that keeps the Setup Assistant from running when the machine first starts up under Lion, or a package that triggers your software installation management system to run, check for, and install any updates on the first boot after OS X is installed.

Additional packages are added to the InstallOSX.pkg by using the `--pkg` option to `createOSXinstallPkg`:

    sudo ./createOSXinstallPkg --source /path/to/Install\ Mac\ OS\ X\ Lion.app --pkg /path/to/LocalAdmin.pkg --pkg /path/to/DisableSetupAssistant.pkg

You can specify multiple packages. They will be installed in the order given.


#### Notes on additional packages

The OS X install environment is very stripped down. There are many command-line tools that are not available when booted into this environment. Python and Ruby are not available, either. This can affect pre- and postflight scripts in packages. You may find that some packages that rely on pre- or postflight scripts to perform important tasks will fail to run properly in the OS X install environment. Check the install log at /var/log/install.log after the install is complete, or open the log window during installation to monitor pre- and postflight scripts.

This issue may limit which packages you can use successfully in the OS X installation environment. You should carefully audit and pre- and post- scripts in any packages you add to your install to be certain they will run correctly in the OS X install environment.

To get an idea what tools are available in the OS X install environment, boot into the Recovery HD. The tools available in this environment are the same as those available in the OS X Install environment.

An additional limitation: the InstallESD.dmg volume has a limited amount of free space. To date, that space has been around 350MB. This is more than enough for some basic configuration/bootstrapping packages. But don't try to add Microsoft Office or iLife or Adobe Photoshop CS6. Not only are they too big to fit in the available space, they all contain pre- and post- scripts that are almost certain to fail in the OS X install environment.

The best approach for additional packages is to add only what is necessary to boot the machine and connect it to your software deployment system -- Munki, Casper, Absolute Manage, etc, and let the software deployment system take over and install everything else once the machine is booted into a full OS.

#### Further note on additional packages and Yosemite (and El Capitan)

Apple has made an undocumented change in Yosemite that affects this tool. If you add any additional packages for installation as part of the OS install/upgrade, they must all be _distribution_ style packages; not component packages.

If you have an existing flat component pkg, you can convert it into a distribution pkg:

`productbuild --package /path/to/component.pkg /path/to/distribution.pkg`

I have not tested bundle-style distribution pkgs; would be interested to know if those are supported as well.

If there are additional packages that the OS X installer does not like, this will result in an error dialog upon reboot:

>Failed to open OS X Installer.<br/>
>The path /System/Installation/Packages/OSinstall.mpkg appears to be missing or damaged.

>with two buttons: "Restart" and "Startup Disk"

The same issue affects customized NetInstall images created with System Image Utility.

If you add additional packages to a customized NetInstall of 10.10 or 10.11, they must be _distribution_ -style packages, or you get the same error.


#### Note on installing OS X on FileVault-encrypted volumes

Installing Lion, Mountain Lion, Mavericks, Yosemite or El Capitan requires a reboot after the install is set up, but before the actual OS X Installer runs. When installing to a FileVault-encrypted volume, after the initial reboot, the pre-boot unlock screen appears. Someone will have to manually unlock the FileVault-encrypted volume before the actual OS X installation can occur. Once the disk is unlocked, installation should proceed normally.  Apple's Install OS X.app does some undocumented (and probably non-third-party-supported) magic to cause an authenticated reboot; this bypasses the pre-boot unlock screen.

If your software deployment tool supports doing an authenticated reboot on Filevault-encrypted machines, you should select that option to avoid this issue. (JAMF Casper is one such tool.)
