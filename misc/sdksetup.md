# Setup LO SDK after building LO from source

Go to $LOROOT/instdir/sdk and run :
```
$ ./setsdkenv_unix
```
hit enter for all questions, letting it use the defaults except for "Automatic deployment of UNO components (YES/NO) [YES]: " type NO.

This will show something like the following after it finishes.

```
 ************************************************************************
 * ... your SDK environment has been prepared.
 * For each time you want to use this configured SDK environment, you
 * have to run the "setsdkenv_unix" script file!
 * Alternatively can you source one of the scripts
 *   "/home/dennis/libreoffice5.3_sdk/dennis-work/setsdkenv_unix.sh"
 * to get an environment without starting a new shell.
 ************************************************************************


 ************************************************************************
 *
 * SDK environment is prepared for Linux
 *
 * SDK = /ssd1/work/dennis/core/instdir/sdk
 * Office = /ssd1/work/dennis/core/instdir/sdk/..
 * Make = /usr/bin
 * Zip = /usr/bin
 * cat = /usr/bin
 * sed = /usr/bin
 * C++ Compiler = /usr/bin
 * Java = /usr
 * SDK Output directory = /home/dennis/libreoffice5.3_sdk
 * Auto deployment = NO
 *
 ************************************************************************
```

As the first section says, **you need to** run following in the shell **before** trying to build our own extensions/UNO standalone programs.
```bash
$ source /home/dennis/libreoffice5.3_sdk/dennis-work/setsdkenv_unix.sh
```

NOTE : *You can also add the above line to `~/.bashrc` so that this line is run automatically every time you start terminal.*
