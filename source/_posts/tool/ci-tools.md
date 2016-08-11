---
title: CI Tools for Exp Deveplopers
categories: tool
tags: [tool,python]
date: 2016-08-11
author: Wayne C.
---

Haijing said that he would allow me to write some blogs in English, because he wanna challenge himself. So, as a considerate coder, I must do as he said.

## CI Process

Everybody must know the process of our 'check in' process.

- Use 'upload.py' to generate a cooder patchset, and it will check whether your code have some style problems.
- Recheck your code.
- 'svn ci' your code with the 'ISSUE=XXX' comment.

## Automatic CI Tool

I've seen Lucas and Haijing copy the issue code and type it in the svn comment. Yeah, it's easy and simple, but I just feel tired about it. After using the GUI 'ci' tools created by CC, I decided to migrate it to the unix shell.

### issue.info

That's an important file of this tool. This file will be generated after executing 'python upload.py', and it contains the issue code of your last cooder patchset.

### Codes

These are the codes.

* Imports

~~~python
import sys
import os
import commands
~~~

* Initializations
    * Get the cooder comment from the shell.
    * Remove the issue.info file.

~~~python
comment = ''
issue_file = 'issue.info'

if len(sys.argv) < 2:
    comment = 'cooder'
else:
    comment = sys.argv[1]

if os.path.exists(issue_file):
    os.remove(issue_file)
~~~

* Executing upload.py
    * Check the result of upload.py.
    * You can change the upload.py position as your wish.

~~~python
(status, output) = commands.getstatusoutput('python /Users/baidu/Documents/Tools/upload.py -y -m "' + comment + '"')
if status != 0:
    notice('execute upload.py failed!', False)
    sys.exit(-1)

if os.path.exists(issue_file) == False:
    notice('execute upload.py failed!', False)
    sys.exit(-1)
~~~

* Get issue code and check in

~~~python
file_object = open(issue_file)
try:
    issue_num = file_object.read()
finally:
    file_object.close()

if int(issue_num) <= 0:
    notice('execute upload.py failed!', False)
    sys.exit(-1)

(status, output) = commands.getstatusoutput('svn ci -m "ISSUE=' + issue_num + '"')
if status != 0:
    notice('svn ci failed!', False)
    sys.exit(-1)

notice('svn ci success!', True)
~~~

* Notice function
    * It can use the different colors to show the notice info in the shell.

~~~python
def notice(string, notice):
    attr = []
    if notice:
        # green
        string = '[WCI NOTICE] @=]=====> ' + string
        attr.append('32')
    else:
        # red
        string = '[WCI ERROR] @=]=====> ' + string
        attr.append('31')
    attr.append('1')
    print '\x1b[%sm%s\x1b[0m' % (';'.join(attr), string)
~~~

## Run the Tool

I will supply the original code for you, and just download it by click [here](/files/ci).
Put 'ci' into a directory, and goto it. Then run the command.

~~~sh
chmod +x ci
~~~

It can make the 'ci' executable. 

Now there a few steps.
- Change your upload.py file path and modify the related part of 'ci'. 
- Add the alias of 'ci' in your '.bashrc' file.
- Goto any module directory, and run 'ci'.

## Extra Knowledge

When we echo some text in the shell, we can add some preffix in front of them to change there colors. There are 8 colors in the Unix/Linux terminal. They're
- Black
- Red
- Green
- Yellow
- Blue
- Magenta
- Cyan
- White

Now you can try a command in your terminal.
~~~sh
echo -e "\033[31m WORD \033[0m"
~~~

It must show a red "WORD", right? Other string around the 'WORD' word control the color of it. You can change the 31 to 30 ~ 37, and they represent different colors.
Actually, we can also change the background of the text. You can just Google it , or maybe we can talk about it next time.