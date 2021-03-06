# HackInOS: 1
[VulnHub link](https://www.vulnhub.com/entry/hackinos-1,295/)  
By Fatih Çelik

* As with [Basic Pentesting: 1](https://github.com/leegengyu/CTF-Walkthrough/blob/master/basic-pentesting-1.md), after the victim VM has been booted up, we are greeted with a login page that requests for the password of user `hummingbirdscyber`. We attempt a few common passwords such as `password` and `admin`, but we do not manage to login as expected.
* The IP address of the victim VM can be found by clicking the icon with 2 arrows pointing up and down respectively on the top right-hand corner of the login page, which is `10.0.2.6`.
![](/screenshots/hackinos-1/vulnerableVMIPAddress.jpg)
* From our Kali VM, run `nmap -p- -A 10.0.2.6`, scanning all ports, and enabling OS detection, version detection, script scanning, and traceroute:
![](/screenshots/hackinos-1/scanAllPortsandServiceVersions.jpg)
* There are 2 ports that are open: `22 (ssh)` and `8000 (http)`. Typically, the HTTP service is run on port 80, but I guess the point of having it on a different port on this vulnerable VM is to test us.
* We also see that there is a `robots.txt` file for the web server that states the disallow entries: `/uploads` and `upload.php`. We will use this piece of information later in the walkthrough.
* *At this point, I am not sure if there is a vulnerability on the SSH service that we can exploit.* Hence, we will go ahead to explore the web service first.

# HTTP service #
* We visit `10.0.2.6` and find a web server running, with the rendered page broken:
![](/screenshots/hackinos-1/wordPressInitialLoad.jpg)
* Hovering our mouse over the `Log in` hyperlink under the `Meta` section shows us that the domain name is expected to be localhost running on port 8000, instead of 10.0.2.6:
![](/screenshots/hackinos-1/httpServicePageHover.jpg)
* `localhost` does not resolve to 10.0.2.6, but to 127.0.1.1 instead - hence the broken page. To resolve this issue, edit our `/etc/hosts` file accordingly:
![](/screenshots/hackinos-1/hostsFileEntries.jpg)
* Note: The commented-out line for the localhost entry that maps to 127.0.1.1 is deliberate so that we can add back this entry after we are done with this challenge, since that was the original entry in the file.
* Reload `10.0.2.6` to see the intended layout of the page. There is only 1 post in the entire WordPress site by user `Handsome_Container`:
![](/screenshots/hackinos-1/wordPressProperLoad.jpg)
* We see that there is a comment on the only post, but the comment reveals nothing of concern:
![](/screenshots/hackinos-1/wordPressPost.jpg)
* Head to the `/wp-login.php` page to confirm that there is indeed a WordPress account with the username `Handsome_Container`. At the same time, we find out that there is no account with the username `admin`.
* The next part which we will explore is the directory `/uploads` and file `upload.php`, which was discovered in our `nmap` scan earlier on.
* Heading to `upload.php`, we see a basic upload page that appears to allow users to upload image files:
![](/screenshots/hackinos-1/uploadPage.jpg)
* Clicking on the `Browse` button, it appears that we are able to upload files of any type.
* I created an empty text file, as well as a second text file with a test line. Uploading these 2 files resulted in a smiley face (literally a `:)`) appearing on the same page. Not too sure what to interpret of this at the moment, though it appears to be a success message indicating a successful upload. However, the files that I uploaded were text files and not image files as was 'required', thus the cryptic smiley face could also mean an unsuccessful upload.
* I grabbed a random `index.png` (not empty) file and submitted it for uploading, where the output is now `File uploaded /uploads/?`. I tried to open `/uploads/index.png` but we get a `Not Found` error message. Thus, it is likely that the uploaded file (if it really was indeed uploaded) is not stored as it is in `/uploads`, at least for its file name.
* I tried to submit a `.jpeg` file for uploading, but the output is now back to a smiley face. There are alot of file image extensions - perhaps this one is not recognised (but I think .jpeg is pretty common).
* Remember the `/uploads` directory that we also saw in the `robots.txt` file earlier? It is likely that the files which we uploaded are found in that directory.
* I clicked on the `Submit` button for a third time without attaching any file to see what would happen, and there is a warning message `getimagesize(): Filename cannot be empty in /var/www/html/upload.php on line 25` on top of the smiley face. From this, the smiley face may not possibly mean a successful upload since it appears when there is no file attached, or it could also mean that the logic flow was badly coded.
* We can also tell that the warning was not deliberately caught by the developer, which explains the raw warning message being the output.
* Looking at the Page Source, we find a comment sneakily hidden at the bottom at line 62 as a comment: `https://github.com/fatihhcelik/Vulnerable-Machine---Hint`. I did not notice it initially because there were only a few elements on the page and I was thus expecting a short source page.
* Loading the GitHub link, we see that there is a README file and `upload.php`.
* From the README, we can deduce that the uploads page was deliberately designed to be vulnerable.
* Opening up the PHP file, we see that there are a bunch of variables being set at lines 18 to 25, after the Submit button is clicked. There is an if-else statement after that, where an upload is only considered successful if the file extension is `.png` or `.gif`.
* All files with with other extensions are not uploaded to `/uploads` and also have the smiley face printed. We have now solved the mystery behind the smiley face. :)
* Notice that line 34 confirms our earlier guess that our uploaded file was not stored as it is in its original form, in terms of the file name: `move_uploaded_file($_FILES["file"]["tmp_name"], $target_file.".".$imageFileType);`.
* Looking back at the bunch of variables setting (especially line 20), we see that our original file name is actually converted to a MD5 hash of the original file name concatenated with a random number between 1 to 100 (both inclusive). For example, using our earlier example of `index.png`, if the number is 100, our new file name is the MD5 hash of `index.png100`.
* Given that the range of random numbers is relatively small, we are able to generate a list of MD5 hashes with each number in the [1,100] range, and check what our uploaded file name is.
* We do this by creating a PHP file (e.g. generateMD5Hashes.php) with the following code:
```php
<?php

for ($i = 1; $i <= 100; $i++) {
	echo md5("index.png" . $i);
	echo "\n";
}

?>
```
* The code acts like a script (using a for-loop) prints a MD5 hash on each line, with the file name and a random number being hashed.
* Note: This code needs to be modified according to your file name. The file name used in this case is `index.png`, since I had uploaded such a file earlier on.
* Next, run `php generateMD5Hashes.php > MD5HashesList.txt`, which redirects the output to a text file instead of the terminal screen.
* Note: Interestingly, inserting the newline ending immediately after the MD5 function (which gives us only 1 line of echo statement) results in only 1 long line of output. Additionally, I got the warning `A non-numeric value encountered in ...`. The PHP version used is `7.3.4-2 (cli)`. *Not exactly sure why this does not work*. Here is a relevant [StackOverflow article](https://stackoverflow.com/questions/42044127/warning-a-non-numeric-value-encountered).
* Next, we wil use `wfuzz` (tool designed for brute-forcing web applications) to find out what is our uploaded file name for `index.png`.
* Run `wfuzz -w MD5HashesList.txt --sc 200 http://localhost:8000/uploads/FUZZ.png`.
* `-w` allows us to specify our wordlist. `--sc` allows us to filter for only responses with the specified code of 200 (i.e. 200 OK).
* On the first column, we see the ID response, which shows that the random number used was 77:
![](/screenshots/hackinos-1/bruteForceResults.jpg)
* Note: Instead of the `--sc 200` option, you can use `--hc 404` (which means to show only responses without the specified code of 404, i.e. Not Found) to get the same result. Also, remove the `--sc/--hc` option if you want to see the entire bruteforce results log (i.e. without filters).
* Upon loading `http://localhost:8000/uploads/ca61202c13182f5fc4021e1b42259c79.png`, I was able to see the image `index.png` file that I uploaded earlier on.
* Since we have confirmed our ability to upload and open an image file onto the web server, we can then upload a malicious file and open it to gain a reverse shell on the web server.
* Go to `/usr/share/webshells/php` and open `php-reverse-shell.php`. Change the values at lines 49 and 50, which is the IP address (`$ip`) and port number (`$port`) respectively of our Kali VM. I will be using port 3000 here.
* However, recall that our file type is restricted to only .png or .gif. One way that servers prevent shells from being uploaded is to restrict the file types of files that users can upload. Nonetheless, we can bypass this limitation by adding the text `GIF89a` or `GIF98` to the start of php-reverse-shell.php.
* Note: Adding the allowed file extensions to our malicious file, e.g. `malicious.php.png` or `malicious.php.gif` does not work because the file type check is done by examining the file MIME (way to identify files according to nature and format) type, and not by splicing the entire string to get the file type.
* After adding the text, upload `php-reverse-shell.php` (we have now bypassed the file extension filter) and then rinse and repeat using `wfuzz` as done previously to obtain the file name of our shell code: `wfuzz -w MD5HashesList.txt --sc 200 http://localhost:8000/uploads/FUZZ.php`. Mine is `6263d635867b1da16ef04b5d419a6fed`.
* Before we open the malicious file on the web server which will initiate the connection back to our Kali VM at port 3000 (or whichever other port you used), we have to `nc` listener on our Kali VM: `nc -l -p 3000`.
* `-l` is used to specify that nc listens to an incoming connection rather than initiating a connection to a remote host. `-p` is to specify the port number. `-v` can also be added if you would like a more verbose output, such as having the message that says that the listener is listening after running the command, versus no such message, i.e. nc is silently waiting.
* After running that command, we now have a reverse shell on the vulnerable web server, as user `www-data`:
![](/screenshots/hackinos-1/successfulReverseShell.jpg)
* Interestingly on some occasions, after we terminate the nc connection, the malicious file appears to no longer be on the web server anymore. We have to re-upload our malicious file to the web server.
* Heading to `/var/www/html`, open `wp-config.php` and we find that the MySQL login credentials are `wordpress:wordpress`. We are not exactly sure if we need this later, but this is one of the pieces of information that I take note once I get access, just in case we are stuck later on (and who knows when it might come in handy).
* Next, we will have to find a way for privilege escalation, using a binary with the setuid bit enabled. Run `find / -user root -perm -4000 -print 2>/dev/null`, just like in mr-robot-1:
![](/screenshots/hackinos-1/suidExecutables.jpg)
* Amongst these binaries, we will use `/usr/bin/tail` to escalate our privileges: `tail -c1G /etc/shadow`, where `tail` allows us to output the final parts of a file:
![](/screenshots/hackinos-1/tailShadowFile.jpg)
* Note: `-c1G` allows us to view the entire file, because the information that we need is found on the first line.
* The encrypted password of `root` is `$6$qoj6/JJi$FQe/BZlfZV9VX8m0i25Suih5vi1S//OVNpd.PvEVYcL1bWSrF3XTVTF91n60yUuUMUcP65EgT8HfjLyjGHova/`.
* We will use `john` to decrypt the encrypted password. Before using john, we will first insert the encrypted password into a stand-alone file, which I have named as `decryptThisPassword.txt`. Next, we will run `john --wordlist=/usr/share/john/password.lst decryptThisPassword.txt`:
![](/screenshots/hackinos-1/johnCracking.jpg)
* Note: The wordlist used is one of those that comes with the installation of `john`. If you were to open the list, it states that the list is based on passwords most commonly seen on a set of Unix system in mid-1990's, sorted for decreasing number of occurences. The last update to the list is in end-2011, with 3546 entries.
* The cracking process is done almost instantaneously, giving us the password as `john`.
* Note: Running the `john` command again will not result in the cracking process being re-run:
![](/screenshots/hackinos-1/johnRerun.jpg)
* Note: If you want to re-run the command to observe the process again, go to `.john` and delete `john.pot`.
* Note: If you want to view the password again without repeating the above process, execute `john --show decryptThisPassword.txt` (replacing the last part with your file name).
* Now that we have our password, we will spawn our interactive shell with `python -c 'import pty; pty.spawn("/bin/bash")'`, then run `su` and enter the password `john`.
* Next, we head to `/root`, and find that there is a file `flag`. However, that file only reveals to us a cryptic message, instead of the actual flag (I think?):
![](/screenshots/hackinos-1/flagInitial.jpg)
* We have already gotten root access, but the flag still eludes us: we will have to find the flag elsewhere.
* Opening up the other files on `/root` directory, the file `.port` holds another cryptic message as well. Have not much of an idea what the `7*` means as well.
* At this point I felt pretty much stuck.
* I went back to the login page of the vulnerable VM (that we had encountered initially), and checked for the username `hummingbirdscyber` against the `/etc/shadow` file, and did not find the username on the list. It seems like the username belongs to an account on another service of the vulnerable VM.
* It seems like there might be a network running within the vulnerable VM, given the hints provided in /root and user `hummingbirdscyber` not being found. Moreover, examining our very first login screenshot, there were 3 tabs under the `Active Network Connections`. Thus far, we would have expected to see only 1 tab - `Wired connection 1 (default)`.
* Running `ipconfig` on our vulnerable VM shows that the internal IP address is `172.18.0.2`:
![](/screenshots/hackinos-1/vulnerableVMNetwork.jpg)
* I tried to run `nmap` to find out the other live hosts on the network but it was not found.
* Remembering that we had a set of MySQL credentials previously, I ran a `ping` request to find out if we can get an IP address of it: `ping -c 1 db`:
![](/screenshots/hackinos-1/pingDB.jpg)
* Note `-c 1` ensures that the ICMP requests would stop after receiving 1 reply, instead of being sent indefinitely.
* The `ping` request resulted in a packet being successfully received - it turns out that the database has its own IP address, which is `172.18.0.3`.
* To log in into the MySQL database, run `mysql -h 172.18.0.3 -u wordpress -pwordpress`:
![](/screenshots/hackinos-1/mySQLLogin.jpg)
* Note: There is no whitespace between `-p` and `wordpress` (the password itself) as per stated in the `man` page. Adding the whitespace will result in MySQL prompting for one immediately even though it was already included in the command.
* Note: While it is required that there is no whitespace for the password, having a whitespace (or otherwise) between `-u` and `wordpress` (the username itself) does not affect the login process, though the `man` page stated that there should be a whitespace.
* If you see `MySQL [(wordpress)]>` after logging in, you can skip the next step.
* After logging in, we find that we see `none` that is part of `MySQL [(none)]>`. This means that we are currently not looking at any databases in particular. Run `show databases;` to find out what databases there are, and then `use wordpress;` to switch to the WordPress database:
![](/screenshots/hackinos-1/mySQLChangeDatabase.jpg)
* Next, we want to see what tables there are within the database, using `show tables;`:
![](/screenshots/hackinos-1/mySQLTables.jpg)
* The very first entry on the top, `host_ssh_cred` stands out from the rest of the tables. Run `select * from host_ssh_cred;` to see what is within the table:
![](/screenshots/hackinos-1/mySQLTablehostsshcred.jpg)
* We are given the password `e10adc3949ba59abbe56e057f20f883e` for the username `hummingbirdscyber`. The password is likely to be a MD5 hash based on `hash-identifier`.
* Using an online password hash cracker, we find that the password is `123456`.
* The `hummingbirdscyber:123456` set of credentials will be used for our SSH login, since the table name suggests that it is the credentials to the host SSH: `ssh hummingbirdscyber@10.0.2.6`:
![](/screenshots/hackinos-1/sshLogin.jpg)
* At this point in time, I found out that there was more than 1 way to escalate our privileges - I will be going through both of them below:

# SetUID Binary for Privilege Escalation #
* I ran the command `find / -user root -perm -4000 -print 2>/dev/null` to find the list of setuid binaries and to see if we could possibly use any of them for privilege escalation:
![](/screenshots/hackinos-1/setuidBinaries.jpg)
* None of the binaries from the list really stood out to me but the very first one did: `/home/hummingbirdscyber/Desktop/a.out`.
* I ran the binary file (`./a.out`), `root` got printed and the program terminated immediately after that.
* Our privileges were very likely to have been temporarily escalated to `root`, before being relegated back to being user `hummingbirdscyber`. When `root` was printed, `whoami` was also likely to have been executed.
* To confirm what we have in mind, run `strings a.out` to see the strings of printable characters in the file:
![](/screenshots/hackinos-1/stringsAOut.jpg)
* Near the start of the output, we see that `setuid` was executed, presumably where we took on the role of `root`, and then `whoami` was executed somewhere not long after. `setuid` was then executed again towards the end of the file, where we were back to being `hummingbirdscyber`.
* Note: The screenshot does not show all of the output from the `strings` command that was executed.
* After that, I was mostly clueless on how to proceed, and this was the part where I learnt: running `$PATH` showed us the order of locations where files would be searched for execution:
![](/screenshots/hackinos-1/pathVariable.jpg)
* In this case, `/home/hummingbirdscyber/bin` was the first directory to be searched.
* This directory did not yet existed, so let us create it using `mkdir`.
* After doing so, the brilliant idea that I learnt is that we will create our own malicious version of `whoami` (which will give us our root shell), which will contain a one-liner `/bin/bash` (or `/bin/sh`), and this version will thus be executed by `a.out`, instead of the standard `whoami`.
* Note: The text editor `vim` is not available, so I used `nano` instead.
* After creating our `whoami`, `chmod +x whoami` to give it permissions to be executable, else running `a.out` would not give us the root shell we desire.
* Finally, run `./a.out` and we have our root shell!
![](/screenshots/hackinos-1/rootBinBash.jpg)
![](/screenshots/hackinos-1/rootBinSh.jpg)
* Depending on whether your one-liner in `whoami` was a `/bin/bash` or `/bin/sh`, your root shell looks slightly differently, where the former's look being very much identical to what we had as a non-root user.
* Note: I could not quite run the `whoami` command to show that we are `root` because doing so would result in another shell being spawn, but you can just run `id` instead to confirm this.
* Head to `/root` directory, and we find the file `flag`:
![](/screenshots/hackinos-1/rootDirectory.jpg)
* Open `flag` and we see an ASCII art that shows a hummingbird:
![](/screenshots/hackinos-1/flagFinal.jpg)
* Hurray, we are done for this exercise!

# Docker Image for Privilege Escalation
* To-be-continued...

# To-be-added subsequently
* Write our hash-generation script using other languages such as Python.

# References
1. https://windsorwebdeveloper.com/hackinos-1-vulnhub-walkthrough/
2. https://medium.com/infosec-adventures/hackinos-walkthrough-3c37844f44a9