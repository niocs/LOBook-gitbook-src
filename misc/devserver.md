# How to setup a LibreOffice development server online for public access

LibreOffice development server is indented to mainly help the Linux newbies to quickly contribute to LibreOffice.

## Hardware
* Processor - Intel i7 4790 3.6 GHz
* Motherboard - ASUS Z97-K
* Hard disk - Seagate 1TB SATA
* RAM - Kingston 8GB x 2

## Internet connection
4Mbps Airtel home connection - goes to 512 Kbps after 120 GB usage.

## Setting up the server
* Install Fedora 22 MATE Compiz Remix
  * 50GB / partition
  * 500MB /boot partition
  * Rest for /home partition
* Install x2go(remote desktop app) server

  `# dnf install x2goserver x2goserver-xsession`
  
* Install iptables

  `# dnf install iptables iptables-services`

* Install sshd

  `# dnf install sshd`
   Change sshd listener port to something else in /etc/ssh/sshd_config

* Install emacs/vim

* Set services to start on boot

  ```
  # systemctl enable sshd
  # systemctl start sshd
  # systemctl disable firewalld
  # systemctl enable iptables
  # systemctl enable x2gocleansessions.service
  ```

* Set hostname

* Setup iptables rules
  If root, allow all traffic
  If non-root allow only :
  ```
  google-dns
  libreoffice.org
  gerrit.libreoffice.org
  bugs.documentfoundation.org
  dev-www.libreoffice.org
  ```

* Install sendmail
  and set it up to send mails through gmail smtp using the instructions in the link [http://linuxconfig.org/configuring-gmail-as-sendmail-email-relay](http://linuxconfig.org/configuring-gmail-as-sendmail-email-relay)

* Configure port forwarding in modem to forward sshd port to server
  In our case it is under "Advanced Setup -> NAT -> Virtual Server"

* Setup IP address change notifier
  Cron a script to email the users whenever external ip adress of the box changes
  for getting ip we used : bot.whatismyipaddress.com


## Setting up the LO dev env in the server

* Install build dependencies
  `# sudo dnf builddep libreoffice`

* Create a folder /home/lo/ as root to host local copy of LibreOffice git code repository.
  and get the source code using :
  `# git clone git://anongit.freedesktop.org/libreoffice/core libreoffice`

* Setup a cron entry to do "git pull" every 1 hour in the folder /home/lo/libreoffice


## What a new user need to do start developing ?

* Request account by sending a mail to **lodevserver@ldcs.co.in**, and we will give you the credentials to login.
* Install x2goclient in your operating system. For Fedora 22+ systems just do `# dnf install x2goclient`.
* Login with the provided credentials. Select "Session type" as "MATE" in the x2goclient settings dialog box.
* Open a terminal in our machine after you login.
* Setup git identity in our machine, as per the following example :

  ```
  $ git config --global user.name "John Doe"
  $ git config --global user.email johndoe@example.com
  ```

* Clone LibreOffice source code repo.
  `$ git clone /home/lo/libreoffice libreoffice`

* Go to libreoffice repo root
  `$ cd libreoffice`

* Build LibreOffice using the following 2 commands ( Second step will take about 1.5 hours when run for the first time )

  ```
  $ ./autogen.sh --with-parallelism=3 --without-junit --enable-debug --without-java --without-doxygen --without-help --without-myspell-dicts
  
  $ make
  ```

* To run LibreOffice which you just now built :
  `$ instdir/program/soffice`

* To start LibreOffice in gdb
  `$ make debugrun`

## How to start contributing ?

* Create a new branch for making your code change :
  `$ git checkout -b branch_name`
  where branch_name can be any string without spaces.

* Select what to work on. If you are new to LibreOffice development, then try an "Easy Hack".
  * Easy hacks are bugs/feature additions in LibreOffice which are comparatively easy to do for a newbie to LibreOffice development.
  * Go to [https://wiki.documentfoundation.org/Development/Easy_Hacks/lists/by_Difficulty](https://wiki.documentfoundation.org/Development/Easy_Hacks/lists/by_Difficulty) for list of easy hacks sorted by difficulty.
  * After deciding on a bug that you want to solve, create a Bugzilla account at [https://bugs.documentfoundation.org/createaccount.cgi](https://bugs.documentfoundation.org/createaccount.cgi) and go the bug's page you are interested and do the following :
    * Click 'edit' next to “Assigned to”, and enter your mail address
    * Set bug status to ASSIGNED
    * Add a comment that you're starting work on this bug
* Now you know what to do, so use your favorite editor - emacs/vim and make the code change.
* Now rebuild LibreOffice using :

  `$ make`
  ( Do this step always after you change code before you run LibreOffice. The time taken to complete this step depends on what you changed. )

  To skip unit tests, use `$ make build-nocheck`
  
  To build just calc, use `$ make sc`
  
* Now test whether your code change worked by running LibreOffice :

`$ instdir/program/soffice`

* If you find issues, and want to debug it, run :

  `$ make debugrun`
  
* If you are confident that your code change worked, then commit your code :

  `$ git commit -m “short_description_of_what_the_code_change_is_about” file1 file2 file3 ....`
  (Where file1 file2 file3 ...etc are the files you modified.)
* Now contact us at **team@ldcs.co.in** and and let us know what you have been working on, then we will instruct you on how to send your patch to LibreOffice developers.

## Source code navigation tool - Opengrok
* Opengrok is web based source code search and cross reference engine. It helps you search, cross-reference and navigate the source code.
* Access it at [http://opengrok.libreoffice.org](http://opengrok.libreoffice.org)

## How to get help when you are stuck ?
* Contact us at **team@ldcs.co.in**
* Go to http://webchat.freenode.net/?channels=libreoffice-dev to chat with LibreOffice developers.
* Post your queries in LibreOffice mailing list : For using this you need to register at [http://lists.freedesktop.org/mailman/listinfo/libreoffice](http://lists.freedesktop.org/mailman/listinfo/libreoffice)

