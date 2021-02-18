---
layout: post
title: VB6 Writing to a Log File
published: true
tags: vb6 log logging file
excerpt_separator: <!--more-->
---


## Overview
I've had to debug several VB6 DLL's during development and found it's invaluable to be able to write to a log to figure out where the processing is getting hung up.

[This forum post gave me the code to do this](http://www.vbforums.com/showthread.php?342619.html)

I'll also copied the code below in case the website above goes away.  Further more, I added a way to append to a file to have a rolling log.  This isn't meant to be used for long term since there is no logic to roll to a new file.
<!--more-->
### An example of reading a file:

``` vb
Dim sFileText as String 
Dim iFileNo as Integer 
  iFileNo = FreeFile 
  'open the file for reading 
  Open "C:\Test.txt" For Input As #iFileNo 
'change this filename to an existing file!  (or run the example below first) 

  'read the file until we reach the end 
  Do While Not EOF(iFileNo) 
        Input #iFileNo, sFileText 
          'show the text (you will probably want to replace this line as appropriate to your program!) 
        MsgBox sFileText 
      Loop 
    
      'close the file (if you dont do this, you wont be able to open it again!) 
      Close #iFileNo 
```
**Note: an alternative to Input # is Line Input # , which reads whole lines.**

### An example of writing a file:

``` vb
Dim sFileText as String 
Dim iFileNo as Integer 
  iFileNo = FreeFile 
  'open the file for writing 
  Open "C:\Test.txt" For Output As #iFileNo 
'please note, if this file already exists it will be overwritten! 

  'write some example text to the file 
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" &  "first line of text" 
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" &   "   second line of text" 
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" &   ""   'blank line 
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" &   "some more text!" 

  'close the file (if you dont do this, you wont be able to open it again!) 
  Close #iFileNo 
``` 
**Note: an alternative to Print # is Write # , which adds commas between items, and also puts the " character around strings.**

### An example of appending to a file 
(use Append keyword instead of Output):
``` vb
Dim sFileText as String
Dim iFileNo as Integer
  iFileNo = FreeFile
  'open the file for writing
  Open "C:\Test.txt" For Append As #iFileNo
'please note, if this file already exists it will be overwritten!
 
  'write some example text to the file
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" & "first line of text"
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" & "   second line of text"
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" & ""  'blank line
  Print #iFileNo, Format(Now, "mm/dd/yyyy hh:mm:ss") & "|" & "some more text!"
 
  'close the file (if you dont do this, you wont be able to open it again!)
  Close #iFileNo
```
