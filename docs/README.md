
# 'REACT' a user-mode rootkit, RAT, and ransomware

_Disclaimer: This project was made for educational and ethical purposes. This article was written
to inform computer users, Software Engineers, Cybersecurity Analysts, and others about how someone can exploit 
multiple vulnerabilities to create a sophisticated piece of malware, and my explanation on how to prevent these vulnerabilities._

## Summary
This project is a remote access Trojan-Ransomware that uses both TCP and UDP sockets for communication.
The payload is inside of a DLL that impersonates Microsoft's
legitimate "mlang.dll" to bypass UAC and escalate privileges when paired with
a System32 mock directory alongside DLL hijacking.

Firstly, the client sends a message over the UDP socket to receive details on 
the TCP server (IP address and port) to establish a connection. This is a unique way to 
change the port and IP address of the command and control center without any
repercussions. If the client doesn't get a response with the TCP server details, the C2 server is
not alive.

Afterwards, the client establishes a connection with the C2 server over TCP and
sends its Windows machine globally unique identifier.
The server attempts to read any client save data if the client was previously
connected to the server by parsing a JSON save file and using this machine GUID as the key.
This means that Client information can be saved. For example, a client's RSA private key 
such as the RSA public and private key pair.
This enforces consistency and ensures that
anything encrypted with is not lost when the C2 server or client disconnects.
In conclusion, any crucial data is stored from a client and loaded on join.
If not, the server saves the data to the JSON file.

The server can request the client to perform actions on the remote host, and the
client responds with details regarding this. Hybrid encryption is used. An AES key is exchanged
alongside client and server RSA public keys.

This write-up explains my experience making this project. 

During this project, I decided to document my experience. This includes the triumphs, roadblocks, ideas I thought of, 
and more that happened during the creation of this project.

## Backstory
Before I started this project, 
[I had previously made a chat room entirely implemented in C for Linux for my Computer Engineering Technology class final](https://github.com/provrb/AMS); 
a basic reverse shell written in C++ for Windows.
I also have been programming in C and C++ while also using the Windows API for over 2 years.

As you may infer, I went into this project with a good understanding of POSIX threads and sockets and the WinSock API.
However, the basic reverse shell I wrote in a month wouldn't even pass an anti-virus heuristic analysis. After researching
the behaviour of anti-viruses, I understood how viruses could evade anti-virus software. One of these was the use of function
pointers, and this is why they will be prevalent throughout the code instead of making direct WinSock API calls. Furthermore,
undocumented Windows API functions that invoke syscalls are also used to further evade detection.

With all this information, I wanted to construct a sophisticated piece of malware for educational purposes 
unlike my previous projects. 
I preferred the idea of ransomware but also the idea of a RAT. 
So I compromised. A RAT to invoke a Ransomware attack with rootkit capabilities and privilege escalation.

### My First Roadblock
Still thinking about this idea of sophisticated ransomware, I realized I hadn't worked with much cryptography, [apart from 'web hacker',
another project on my GitHub where I decrypted Chrome and Firefox cookies](https://github.com/provrb/web-hacker), passwords, history, etc, using AES, Base64, and managing SQL
Databases. The idea of encrypting files with a key that couldn't be reverse-engineered stumped me until I researched further into cryptography about RSA encryption.

There is such thing as symmetrical and asymmetrical encryption algorithms. An example of symmetrical encryption would be the popular
AES algorithm, while an example of asymmetrical encryption would be RSA.
Symmetrical cryptography algorithms work to generate a key of a set size that can both decrypt and encrypt data. Asymmetrical cryptography algorithms
work to generate a pair of keys, one 'private' and one 'public' where the public key is used to encrypt data and the private key is used to decrypt encrypted data.
However, RSA encryption does have limitations when it comes to this scenario, and that is the size of the data to encrypt cannot be bigger
than the key size in bytes. For example, RSA 1024 bit keys can only encrypt 128 bytes of data at a time.
Since my packets are 8kb in size, RSA won't work to encrypt them, so a much more complicated scheme is required.

An RSA key pair is generated when the server starts, these will be the encryption keys for the session. An AES key is
generated each time a client connects on the server. On the client, another RSA key pair is generated. Once connected,
the client and server handshake to exchange information such as the clients RSA public key, the servers RSA public key and finally
the AES key which is sent from the server, encrypted with the clients public key, and then decrypted by the client with its private key.
AES is particularly useful for sending encrypted requests and encrypting files as there are no size limitations.
This approach is called hybrid encryption, the process of combining asymmetrical encryption algorithm RSA, and symmetrical
encryption algorithm AES to securely protect data as well as secret keys. This approach is especially useful when encrypting files.
To encrypt a file, I would generate a unique AES key for the file, encrypt the file using the AES key, encrypt the AES key with an RSA public key sent
from the server and append the encrypted AES key to the file as well as the IV (initialization vector).

In conclusion, I successfully implemented a robust file encryption process utilizing hybrid encryption, 
ensuring that only the appropriate decryption function can recover the data when provided with the necessary private key. 
While this discussion is purely hypothetical, the described mechanism demonstrates the strength of combining RSA for secure key exchange and AES for efficient bulk encryption. 
Such encryption schemes are widely applicable for secure data protection but also highlight the potential misuse of cryptography, as seen in ransomware scenarios.

## File format
After brainstorming and thinking about all these methods, I proposed myself with the question: Which file format would I want to approach this project with?
A regular EXE? An ISO file? Maybe even a .SYS file. I started digging deeper into how DLLs work,
how to write one in C++, and how can they be used for malicious intent. That is when I discovered a vulnerability that involved DLLs and System32.
Both are fascinating subjects, which ultimately influenced my decision to include the payload in a DLL. 

## DLL Hijacking
DLL hijacking is a vulnerability where an application loads a malicious DLL rather than the intended DLL and thus 
can execute malicious code in the background while everything appears normal to the user. 

Now the question is why does this work?

When an application loads a DLL, Windows searches four different regions in this order to find a DLL matching the requested name:
1. The directory the application is in
2. System32
3. The current working directory
4. System paths set in the environment variables panel

This function is built into the Windows OS as a standard to search for requested DLLs.

The first DLL that is found will be loaded. If a process that has Administrator permission loads a DLL, that DLL will have Administrator permission.
Similarly, if a process has SYSTEM privileges and loads a DLL, that DLL will have SYSTEM privileges, and so on. A DLL can elevate permissions without
asking the user and can bypass UAC, but we'll cover that next.

One real-world example of where DLL hijacking was prevalent was the popular 'Stuxnet' worm discovered in 2010. This worm
exploited a total of FOUR zero-day exploits and leveraged DLL hijacking as a targeted attack toward Iranian nuclear facilities.

## Mock Directories
How does System32 correlate with DLL hijacking? In Windows, it is possible to make something called a "mock directory", where an empty character is present after a folder name.
For example, the System32 folder path is "C:\Windows\System32". We can mock the System32 directory by adding a space after "Windows". 
Using this method, this is what the mock Windows directory would be like "C:\Windows \System32". 
However, by using File Explorer, trying to create a folder with a space at the end will just remove the space. Instead
of using File Explorer, we can programmatically create this mock directory by using PowerShell or C++. Try it yourself! Simply input this command into a new
PowerShell process:

New-Item "\\?\C:\Windows \System32" -ItemType Directory

Interestingly, Windows treats this fake System32 directory just like the real Windows folder!
Once you make that mock directory, look in your new Windows folder, it will also contain every single folder and file that's in the real Windows folder.

You can't make a text document in that folder or delete it from File Explorer, you must use PowerShell again. To
get a better understanding, I recommend that you try it for yourself, it will help when trying to 
wrap your head around this information.

## Using Mock Directories with DLL Hijacking to Bypass UAC
Using the information we learned about mock directories and DLL hijacking we can put these methods together to bypass
Windows User Account Controller. First of all, we need to search for System32 executables that auto-elevate to Administrator privileges,
we can use a repository posted on GitHub that describes all authentic System32 executables that will auto-elevate the respective DLLs.

https://github.com/wietze/windows-dll-hijacking/blob/master/dll_hijacking_candidates.csv

*In my case, I chose to exploit ComputerDefaults.exe and mlang.dll.*

I started with a blank DLL template that would run when a process attached to it, and I inserted ComputerDefaults.exe into my System32 mock directory. 
When I ran the EXE, I received a message showing that my version of MLANG.dll was loaded.

Doing all of this though would break the actual executable because the whole reason the DLL is getting loaded is because the executable
needs code from our DLL. To fix this, I told the linker to link all the functions as exports ComputerDefaults requested from us.
After, I loaded up PE-Bear to find what exports the authentic mlang.dll provided and copied all function names to my linker commands, though
I redirected the export to the path of the real mlang.dll so ComputerDefaults was able to use the real functions but also simultaneously run my DLL.

By making our DLL's export table identical to the authentic MLANG.dll, the executable requiring the DLL wouldn't break.
When the real ComputerDefaults.exe is executed, it would auto-elevate my DLL to Administrator privileges, bypassing the Windows UAC,
and allowing me to run malicious code in the context of an Administrator.

## Process Manager
One of the first things I started to work on after dllmain.cpp was finished, was the Process Manager library. Here I created an interface for
loading different functions by getting the current processes PEB address and iterating over each LIST_ENTRY of the module
list until I found the library I was looking for. This would allow me to dynamically load libraries but, I would still need
to get the function address to actually import functions from the library. I started to research this area and found
that it was relatively easy to get the exports of a library by finding the export directory from the headers, all that was
left was to compare function names to the export I wanted. By iterating over the NumberOfNames field in the export directory
I can check the array of all the export names to my desired function, once I found the function I wanted, I would just
return the absolute address of the function. 

After a couple of these functions, loading functions would be convenient as I wouldn't need to pollute my namespace or import table,
which is one way to attempt to evade AVs. Afterwards, I got started on trying to elevate my permissions even more.

## SYSTEM Permissions
In Windows, there is a built-in account known as SYSTEM or LocalSystem. This account is used by the operating system and
services which require higher privileges to manage important resources and processes. Luckily for threat-actors
but unlucky for users, these privileges aren't too difficult to obtain and can allow a program to execute malicious instructions
with these obtained privileges. By using my amazing and totally convenient procutils library, I was able to obtain a "token" to 
acquire these SYSTEM privileges. With this token, I can open and perform commands in the LocalSystem context. To obtain this token
we first got to take advantage of a process running with SYSTEM permissions. A known process in this category is winlogon.exe. 
This application is responsible for handling the login or logoff of a user, and securely authenticating said users. Winlogon.exe is
also used for handling the secure attention sequence, (pressing CTRL + ALT + DEL). One reason why this process is also to exploit
is because it runs at startup, runs all the time, and runs before the user logs in. Meaning this process will always be running!

To take advantage of winlogon.exe, we need to find the process ID of it. By taking a snapshot of all currently running processes
and iterating through them while comparing the name to the process we want, we can easily retrieve the process idea of Winlogon.
The next process is extremely tedious and involves getting a process's security token. A security token is pretty much a piece
of information identifying which permissions a process has.

To start, we need to open a handle to the Winlogon process using the PID acquired before. In this procedure, I took advantage of undocumented
syscalls for the Windows operating system. Which you'll see.

Firstly, We can use the NtOpenProcess undocumented syscall to retrieve a handle. 
Using this newly acquired process handle, we need to get the security token of that process. I used NtOpenProcessToken,
another undocumented syscall, along with the TOKEN_DUPLICATE permission, saying that we want permission to duplicate this security token. This
is allowed because we're running in the administrator context. With the security token for winlogon.exe, we can safely duplicate it by using NtDuplicateToken
and return a HANDLE to a SYSTEM token! With this token, we can execute commands with SYSTEM permissions.

But even though SYSTEM is almost equivalent to kernel-level control, is there any way to get a higher privilege context?

## Trusted Installer Permissions
You've probably seen something related to Trusted Installer on your computer before. Perhaps you tried deleting a system
file and were greeted with a message like "You require permission from Trusted Installer". Trusted Installer is a user account
like you, but it has the highest permissions available in user mode, higher than the previous group, SYSTEM. Windows has this group built-in to prevent damaging important
system files. My hypothesis: Is it possible to elevate to Trusted Installer privileges, considering how easy it was to obtain Administrator and SYSTEM privileges?

During this procedure, I used two main functions from the ProcessManager header file. In particular, StartWindowsService and CreateProcessAccessToken use the same
function to obtain a SYSTEM token. You may see where this is going.

Rather than being an actual running process like winlogon.exe, TrustedInstaller is embedded into Windows as a service and thus has to be started.
To start any Windows process, we first need to obtain a handle for the Windows service control manager, a special process responsible for starting and stopping.
Windows processes, so we can start and stop Windows processes. After, we need to obtain another handle for the service by opening it with the SC manager.
Though, we do not know if this service is running, or pending start or a stop, so we must query the status of the service. Once the service is running, we can take the
process ID of that service, and run it through CreateProcessAccessToken to return a security token in the context of the provided process, in this case, the Trusted
Installer service.

In conclusion, we got the PID of the Trusted Installer service, started it using the Windows SCM, and duplicated the security token for that process as we did
with winlogon.exe. Using the handle of the security token, I can run and manipulate processes in the TrustedInstaller context. The highest possible User-Mode privileges
implemented in the Windows operating system. Meaning this has the potential to become a User-Mode root kit as we have persistence, privilege, and stealth.

## How could this malware be avoided; as you-the user?
Overall, there is no silver-bullet for ransomware, rootkits, or RATs. However, there are some steps you can take to prevent these types of malware from infecting your computer.

1. Update software regularly 
   - One of the main methods of remaining safe is to keep your computer up-to-date. 
     This includes updating your operating system, software, and drivers.
     While this is not totally convienant, this can prevent against malware that take advantage of known vulnerabilities in these
     pieces of software, which can be detrimental to your computer or a network.
2. Use a good antivirus program.
   - Although the idea of whether or not an anti-virus program can be useful is a controversial topic
     they can help protect your computer from known malware, especially as the average user.
     
3. Be careful browsing the internet
   - Only download files from trusted sources and be wary of files from unknown sources.
     Downloading unknown files is a leading cause to compromised networks and computers.
4. Be careful when clicking on links
   - A popular method to compromise networks when a malware cannot directly be downloaded is through phishing and social engineering.
     Social engineering is the act of manipulating people into performing actions or divulging confidential information.
     This can be paired with phishing, where a user is tricked into clicking on a malicious link.
     To prevent against this, make sure you know the sender of the email and the link you are clicking on and 
     100% sure of its legitimacy.

Although these steps can help prevent malware from infecting your computer, it is important to remember that no method is foolproof.
Sometimes theres nothing we can do, for example if a zero-day exploit is abused malware may be able to spread without any
user interaction. It is important you try your best to prevent malware from infecting your computer.
   
## Conclusion, what I learned
From this project, I learned a lot about the Windows operating system, the Windows API, and how to exploit vulnerabilities in the Windows operating system.
I learned how to use undocumented syscalls to elevate my permissions to SYSTEM and TrustedInstaller, and how to use DLL hijacking to bypass UAC.
I also learned how to use hybrid encryption to securely encrypt files and data, and improved my understanding on sockets.

Most importantly, through this project I was able to think like a threat actor. To know a threat-actor is to be a 
good threat-hunter. I'm able to understand how to prevent exploitation of vulnerabilities. 
From a more-cybersecurity oriented perspective, this would be a great experience 
I can refer to when blue-teaming, rather than red-teaming.

This project was incredibly fun to create and I hope you enjoyed reading my first write-up.
I hope to make more projects similar to this as well as projects to go against threat-actors.

Thank you for reading.
