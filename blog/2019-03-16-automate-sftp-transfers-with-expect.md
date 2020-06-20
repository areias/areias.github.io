---
layout: post
title:  "Automate SFTP file transfers with Expect"
date:   2019-03-16 12:10:49 -0500
categories: linux 
---


I currently have an ongoing experiment running on a work server - on a side note, is there a better feeling than surfing or having beers with friends while somewhere your code is running generating results?

So once a day I want to download the latest results to look over (with a feeling of almost christmas-like antecipation, yes I'm weird). This is rather cumbersome to do manually; on the server side I need to upload new results to the SFPT server, and then I have to download the files from the SFPT server into my local machine in order to analyse them.

Each day there are maybe 20 new result files, and in total there are hundreds so I want to transfer just the new files. 

SFPT has a **-b batch** option that allows you to send in a series of put/get commands in a script but this doens't work when you're using an interactive authentication method like I am. I *could* ask the server admin to configure ssh access but every request I make takes a while (they're great, they just have a lot on their plate) so I try to avoid doing that when possible.

Turns out I can automate the uploading/downloading of new files from the server everyday using [expect](https://linux.die.net/man/1/expect) and a [cron](http://man7.org/linux/man-pages/man8/cron.8.html) job.

## Install Expect
If you don't have expect installed you can:
```bash
sudo apt install expect
```

The simplest [expect script](https://likegeeks.com/expect-command/) to automate loging into an SFTP server and uploading a file looks like:
```bash
#!/usr/bin/expect 
spawn sftp user@server
set timeout 120
expect "Enter your PASSWORD:"
send "myPassword\r"
expect "sftp>"
send "cd path/tofile\r"
expect "sftp>"
send "put filename\r"
expect "sftp>"
send "bye\r"
expect eof
```

The default timeout for expect is 10 seconds, but depending on the size of the file you're uploading you might want to increase it otherwise it will cut out before the upload is finished. 

You then call the script by typing:
```bash
usr/bin/expect myScript.sh
```

On the server I can create a list of new result files created in the last 24 hours:
```bash	
find . -mtime -1 > newFiles.txt
```

Then I can manipulate this list using [sed](https://www.gnu.org/software/sed/manual/sed.html) to create an expect script that will log into the SFTP server and issue a series of put requests, one for each file.

First I remove first line that consists of just a period.
```bash
sed -i -e '1d' newFiles.txt
```

I add send and expect prompts to the beginning and end of each line;
```bash
sed -i -e 's/^/send "get /' newFiles.txt
sed -i -e 's/$/\\r"\nexpect "sftp>"/' newFiles.txt
```

and SFTP login details to the beginning of the file.
```bash
sed -i -e '1s/^/spawn sftp user@server \
set timeout 3600 \
expect "Enter your PASSWORD:" \
send "myPassword\\r" \
expect "sftp>" \
send "cd path\/tofile\\r" \
expect "sftp>"\n/' newFiles.txt
```
Note the escaping of backslashes in lines 4 and 6 above!

And finally, I append as last two lines:
```bash
echo 'send "bye\r"'>> newFiles.txt
echo 'expect eof' >> newFiles.txt
```

So now your newFiles.txt can run as a expect script itself to upload each of the new files to the SFTP server.
```bash
/usr/bin/expect < newFiles.txt
```

So all the above code goes into one bash script that gets run everyday at 5 am using cron.


With minor tweaks I can use the same logic to automatically download the new files locally everyday at 6 am (one hour in between to allow for downloaing of large files). That means at 7 am I can already look over what new surprises my experiments may hold over a cup of coffee!

The bash script to download results locally:
```bash
#!/usr/bin/bash

# sftp to get newFiles.txt
/usr/bin/expect -c ' 
	spawn sftp user@server
	set timeout 120
	expect "Enter your PASSWORD:"
	send "myPassword\r"
	expect "sftp>"
	send "cd path/tofile\r"
	expect "sftp>"
	send "get newFiles.txt\r"
	expect "sftp>"
	send "bye\r"
	expect eof'

# maniuplates newFiles into an expect script
sed -i -e '1d' newFiles.txt
sed -i -e 's/^/send "get /' newFiles.txt
sed -i -e 's/$/\\r"\nexpect "sftp>"/' newFiles.txt
sed -i -e '1s/^/spawn sftp user@server \
set timeout 3600 \
expect "Enter your PASSWORD:" \
send "myPassword\\r" \
expect "sftp>" \
send "cd path\/tofile\\r" \
expect "sftp>"\n/' newFiles.txt
echo 'send "bye\r"'>> newFiles.txt
echo 'expect eof' >> newFiles.txt

# use newFiles to get new files 
/usr/bin/expect < newFiles.txt
```
