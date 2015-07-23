# How to setup a libreoffice development system

## Requirements
* You must have at least 26 GB of disk space for getting the git repo code and building it and a few more GBs for a debug build.
* Need a minimum of 4 GB RAM but more than 8 GB is ideal.

### Reference system spec
* RAM - 16 GB
* Disk - 256 GB SDD
* CPU - Intel(R) Core(TM) i7-3770K CPU @ 3.50GHz


## Build dependencies
On Fedora 15+ & derivatives upto 21

`sudo yum-builddep libreoffice`

On Fedora 22 upwards :

`sudo dnf builddep libreoffice`

## Getting the LO source code
The following gets the LO source code from git repo. This will take some time depending on your bandwidth.

`git clone git://anongit.freedesktop.org/libreoffice/core libreoffice`

## Build from source code
Go to the root of the downloaded git repository.
```
./autogen.sh --with-parallelism=5 --without-junit --enable-debug  --without-java  --without-doxygen --without-help --without-myspell-dicts
make

```
You can tweak the --with-parallelism param value to the number of cores you want to use for compilation.
--enable-debug adds debugging symbols for debugging later.

On our reference machine, make takes 1.75 hrs to finish.

## Running Libreoffice
To run the LibreOffice you have just built, do

`$ instdir/program/soffice --calc`


## Advanced setup
Please refer [https://wiki.documentfoundation.org/Development/BuildingOnLinux](https://wiki.documentfoundation.org/Development/BuildingOnLinux)

## Debugging
The easiest way to start LO in gdb is to run :
`make debugrun`

Refer [https://wiki.documentfoundation.org/Development/How_to_debug](https://wiki.documentfoundation.org/Development/How_to_debug) for a primer on using gdb

## Setting yourself up for patch submission
Libreoffice developers use a code review tool called [gerrit](https://gerrit.libreoffice.org). Refer the LO wiki [https://wiki.documentfoundation.org/Development/gerrit](https://wiki.documentfoundation.org/Development/gerrit) for setting up your gerrit account and steps to patch submission.
For further information on patch submission and modification during review refer [https://wiki.documentfoundation.org/Development/gerrit/SubmitPatch](https://wiki.documentfoundation.org/Development/gerrit/SubmitPatch)

## Libreoffice Opengrok
[Libreoffice Opengrok](http://opengrok.libreoffice.org) is a very useful tool to search in the libreoffice git repositories.
