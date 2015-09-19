# rctrlx

rctrlx allows commands to be executed on remote Windows hosts


## Usage

```
rctrlx [hostname] [OPTIONS] /app [Application executable parameters]
/app            Specifies the application to run.

Options list:
/c file1,file2  Copies a list of files to the remote working directory.
                These files will be removed on the server exiting successfully
                unless the /d switch is used.
                Note1: The windows directory is the default working directory
                Note2: This first copies to the system32 directory and
                then gets moved to the working directory if specified
                Note4: This will overwrite existing files

/d              Doesn't wait for the application to exit

/i              Displays on the interactive desktop

/s              Run as local system

/u              Username to access the remote host and execute the
                application in the format Domain\Username or just Username

/p              Password for the above username, you will be
                prompted if a username is specified without a password

/w              Working directory on the remote machine

/rrc            Returns the remote applications status on exit

/ru             Remote username that the remote service will run under

/rp             Password for the above remote username, you will be
                prompted if a remote username is specified without a password

/cm XxY         Compatibility mode for applications that create subshells
                e.g. telnet, ftp, powershell etc. The XxY value specifies the
                buffer size of the remote command prompt only when output is
                redirected e.g 80x300. If output is sent to the screen this
                value can be 0x0. See the readme for more information

/cp CPI,CPO     Sets the code page used for stdin and stdout (from the remote
                hosts perspective) in the format CPSTDIN,CPSTDOUT
                e.g. 65001,65001. See the readme.txt for more information

/m              Run in maintance mode which is an extra connection that
                can be used to wait on a command that will terminate hung
                applications. This connection will be disconnected after
                a short period of time to prevent it from also hanging.
                This option is not compatible with the /cm switch
```

* Usernames and passwords are transferred in cleartext
* Please check the remote host's event logs if errors occur

## Example usage: working directory

Copy the local file c:\somefile.txt to c:\temp on the remote host and then open it with notepad:

    rctrlx hostname /w c:\temp /c c:\somefile.txt /i /app notepad.exe somefile.txt

## Example usage: copying files

Copy a file using an absolute path:

    rctrlx hostname /c c:\somefile.txt /app notepad.exe

Copy a file using a relative path. Note that `somefile.txt` must be in the same directory as the rctrlx executable:

    rctrlx hostname /c somefile.txt /app notepad.exe

## A typical event log message on the remote host:

> The description for Event ID ( 1 ) in Source ( Application ) cannot be found. The local computer may not have the necessary registry information or message DLL files to display messages from a remote computer. You may be able to use the /AUXSOURCE= flag to retrieve this description; see Help and Support for details. The following information is part of the event: Error transferring file FilterLog.log: The system cannot find the path specified.

The only relevant part of this message is the last sentence:

> Error transferring file FilterLog.log: The system cannot find the path specified.


## Compatibility mode

When using compatibility mode, the XxY value must be specified but is only actually used when I/O is redirected. If the output is being sent directly to the screen, 0x0 can be used. When I/O is being redirected, the XxY value must be between 1x1 and 9999x9999. Note that the remote buffer cannot be set to anything less than the default (80x25 on my system), and setting it to the maximum value will cause significant delay. Even a value such as 80x9999 will take roughly 30 seconds to complete.

Because of the way compatibility mode works, it is only possible to redirect output that fits within the remote console's buffer, i.e., if the buffer is set to 80x9999 only 9999 rows of data, which are each 80 columns wide, will be displayed.


## Specifying the code page

Valid code pages include all those specified on the Code Page Identifiers table as found on MSDN (search for "Windows Code Page Identifiers"), as well as those mentioned below.

```
Num|Name___________|Description_________________________________________________________
0  | CP_ACP        | The system default Windows ANSI code page.
1  | CP_OEMCP      | The current system OEM code page.
2  | CP_MACCP      | The current system Macintosh code page.
3  | CP_THREAD_ACP | Windows 2000: The Windows ANSI code page for the current thread.
42 | CP_SYMBOL     | Windows 2000: Symbol code page.
----------------------------------------------------------------------------------------
```

## Troubleshooting

**Q**: How do I access the remote host's event logs?

A: Get a terminal up on the remote host and run `eventvwr`, or execute `mmc` (Microsoft Management Console) and navigate to file -> add\remove snap in -> add -> event viewer, then specify the remote host in the "another computer" text box. Be sure to close the mmc console before using the /u option

**Q**: I get the error: "Unable to connect to the remote host: Multiple connections to a server or shared resource by the same user, using more than one user name, are not allowed. Disconnect all previous connections to the server or shared resource and try again."

A: Close any open connections to the remote host including Windows shares, etc

**Q**: In the remote host's event logs I get the error: "Error transferring file somefile.txt: The system cannot find the path specified."

A: The working directory probably doesn't exist or is invalid, e.g. you don't have access, etc

**Q**: How can I manually remove the server service on the remote host?

A: On the remote host, stop the service `rctrlx` and then run `sc delete rctrlx`

**Q**: How can I force the service on the remote host to stop?

A: Kill the rctrlxsrv.exe executable. `sc delete rctrlx` will then remove the service

**Q**: Why is the return code always 0 even when an error occurs?

A: Make sure that the environment variable `ERRORCODE` has not been explicitly set (use the set command to show all env vars)

**Q**: I'm trying to see output from powershell, ftp, telnet, or other similar programs, and they aren't working as expected

A: Applications that use subshells are only supported through the use of the /cm switch

**Q**: I've executed a program that uses subshells without the /cm switch (like telnet, powershell, etc), and can't create any new rctrlx sessions on the remote machine

A: Manually kill the remote process that's using a subshell

**Q**: I'm running a Windows Vista-based operating system (Windows Vista, Windows Server 2008, etc) and I keep getting an access denied permissions error

A: Rctrlx must be able to access the `admin$` share, which is disabled by default. To enable it, edit the registry and navigate to `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion \Policies\system\`, then add a DWORD value of `LocalAccountTokenFilterPolicy` and set the value to 1. Restart the system

**Q**: I'm trying to remotely execute `cmd.exe /u` so that I get Unicode output but it's getting truncated

A: See below and also note that many commands don't actually seem to output Unicode. In many cases, the output will consist of nothing more than question marks. This can be observed by executing `ipconfig`

**Q**: I'm trying to silently uninstall an MSI, e.g. using `msiexec /x "{PRODUCT_CODE}" /QB`, or perform a similar operation, and get truncated output or spaced text as in the following "T H E  O P E R A T I O N"

A: Change the code pages via `/cp 0,1200`. This will cause any input to the remote process to be re-encoded as ANSI, and the output will be interpreted as UTF-16

**Q**: I'm running the command prompt, or other application, remotely and when I send any command I get the response "more?"

A: You must set the appropriate code pages using the `/cp` switch

**Q**: I'm setting a code page that I know exists but get the error message: Invalid code page specified
A: This is most likely due to the code page not being installed. If you go to the advanced tab under regional and language settings from control panel, you should be able to select it

**Q**: I'm entering non-English characters and only see a question mark instead of the expected symbol

A: This is most likely because the console font you have selected cannot represent the character entered. There are 3 solutions:

* Change the console font to something like Lucida Console from the font tab under console properties
* Add an installed font to the console font list by editing the registry (Search: "Windows + add console font") 
* Install a third-party font

## Contact

Please send bug reports, questions, etc, to `gtaLon51@gmailNOSPAMTHXS.com`

