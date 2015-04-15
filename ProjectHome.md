## Description ##
The SSH Power Tool (sshpt) enables you to execute commands and upload files to many servers simultaneously via SSH _without_ using pre-shared keys.  Uploaded files and commands can be executed directly or via sudo.  Connection and command execution results are output in standard CSV format for easy importing into spreadsheets, databases, or data mining applications.

### Advantages ###
Since sshpt does not utilize pre-shared SSH keys it will use provided credentials for all outgoing SSH connections.  This has many advantages:

  * **Can be used immediately:** No need to spend enormous amounts of time setting up pre-shared keys.
  * **More secure:** A server with pre-shared keys is a huge security risk.  It literally holds the keys to the castle!  With sshpt you can perform all the same tasks as with pre-shared keys with less risk to your infrastructure.
  * **More compliant:** Executing commands as root via pre-shared keys makes it hard to figure out after-the-fact who did what (root as a shared account).  When an administrator (or user!) uses sshpt to execute commands it is much easier to figure out "who did what and when" from an auditing standpoint.

### Requirements ###
The SSH Power Tool requires Python 2.5+ (not 3.0+ yet) and the following Python modules:
  * [Paramiko](http://www.lag.net/paramiko/) - SSH implementation in Python.
  * [pycrypto](http://www.amk.ca/python/code/crypto.html) - Python Cryptography Toolkit (required by Paramiko).

### Also A Module ###
sshpt is designed to be easily imported as a module so that any Python application can quickly and easily be given the ability to SSH into a large number of hosts to gather information or perform administrative tasks.

### Report Format (CSV) ###
Execution results are reported in [RFC 4180](http://tools.ietf.org/html/rfc4180)-compliant CSV format...
```
"host","connection result","datetime","command","command output"
```
  1. "host" - IP or hostname of the host in question.
  1. "connection result" - reports SUCCESS if sshpt was able to successfully connect and authenticate to the host in question, FAILED otherwise.
  1. "datetime" - The exact date and time the command was executed on the host.
  1. "command" - The command that was executed.
  1. "command output" - The output of the executed command.  Right now this is just stdout but adding support for recording stderr is in the TODO list.

In the case that multiple commands were passed to sshpt the report will prepend '_index_: ' before each command and each result respectively.  Example:

```
./sshpt -f ../testhosts.txt "echo foo" "echo bar"
Username: myuser
Password:
"devhost","SUCCESS","2009-02-20 16:20:10.997818","0: echo foo
1: echo bar","0: foo
1: bar"
"prodhost","SUCCESS","2009-02-20 16:20:11.990142","0: echo foo
1: echo bar","0: foo
1: bar"
```

## Use Cases ##
The following are examples of how sshpt might be used.
### Run Commands ###
Your boss has asked you to provide a list of Linux servers that have a certain package installed (e.g. wget).  You don't have root access to all systems but you do have the ability to log in...

**IMPORTANT:** Always make sure to surround your commands in quotes!
```
$ ./sshpt.py -f hostlist.txt -o wget_report.csv "rpm -q wget"
Username: myuser
Password:
"devhost","SUCCESS","2009-02-19 15:07:57.078446","rpm -q wget","wget-1.9.1-17"
"prodhost","SUCCESS","2009-02-19 15:07:58.663611","rpm -q wget","Package wget is not installed"
"blah","FAILED","2009-02-19 15:07:59.134068","rpm -q wget","(-5, 'No address associated with hostname')"
...
```
In this example the results were output to stdout and to an outfile (-o), wget\_report.csv.

### Upload Files ###
You've just got some new DNS servers in and you need to configure all of your servers to use them.  Your task is to replace the existing resolv.conf on all of your servers with a modified one...
```
$ ./sshpt.py -f hostlist.txt -u myuser -s --copy-file=resolv.conf --dest=/etc/resolv.conf
Password:
"blah","FAILED","2009-02-19 15:20:14.134068","sudo -u root sshpt: sftp.put /etc/resolv.conf blah:/tmp/resolv.conf)","(-5, 'No address associated with hostname')"
"devhost","SUCCESS","2009-02-19 15:20:15.401703","sudo -u root sshpt: sftp.put /etc/resolv.conf devhost:/tmp/resolv.conf)","-rw-r--r-- 1 root root 72 2009-02-19 15:20 /tmp/resolv.conf"
"prodhost","SUCCESS","2009-02-19 15:20:15.691474","sudo -u root sshpt: sftp.put /etc/resolv.conf prodhost:/tmp/resolv.conf)","-rw-r--r-- 1 root root 72 2009-02-19 15:20 /tmp/resolv.conf"
...
```
This example is kind of complicated so here's a detailed rundown of what happened:
  1. The file, resolv.conf was copied to each host in the hostlist.
  1. Since the sudo switch (-s) was used the file was SFTP'd to /tmp and then copied to the supplied destination (--dest) using the sudo command ("sudo -u _user_" is automatically prepended to the CSV output when the sudo switch is used).
  1. Since a username was supplied on the command line (-u) sshpt only asked for a password.

Note: If the sudo switch (-s) was not set the file would have been copied directly to the supplied destination (and owned by the username that was used to connect).  In this case the command output would have reported "Permission Denied" since /etc/resolv.conf can only be edited/replaced by root.

## Command Line Help ##
### SSHPT Help Text ###
```
./sshpt.py --help
Usage: sshpt.py [options] [command] [arguments...]

Options:
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  -f <file>, --file=<file>
                        Location of the file containing the host list.
  -o <file>, --outfile=<file>
                        Location of the file where the results will be saved.
  -a <file>, --authfile=<file>
                        Location of the file containing the credentials to be
                        used for connections (format is "username:password").
  -t <int>, --threads=<int>
                        Number of threads to spawn for simultaneous connection
                        attempts [default: 10].
  -p <port>, --port=<port>
                        The port to be used when connecting.  Defaults to 22.
  -u <username>, --username=<username>
                        The username to be used when connecting.
  -P <password>, --password=<password>
                        The password to be used when connecting (not
                        recommended--use an authfile unless the username and
                        password are transient
  -q, --quiet           Don't print status messages to stdout (only print
                        errors).
  -c <file>, --copy-file=<file>
                        Location of the file to copy to and optionally execute
                        (-x) on hosts.
  -D <path>, --dest=<path>
                        Path where the file should be copied on the remote
                        host (default: /tmp/).
  -x, --execute         Execute the copied file (just like executing a given
                        command).
  -r, --remove          Remove (clean up) the SFTP'd file after execution.
  -T <seconds>, --timeout=<seconds>
                        Timeout (in seconds) before giving up on an SSH
                        connection (default: 30)
  -s, --sudo            Use sudo to execute the command (default: as root).
  -U <username>, --sudouser=<username>
                        Run the command (via sudo) as this user.
```

### Notes On Usage ###
Always make sure to put your commands in quotes like so:
```
./sshpt.py -f hostlist.txt "ls -l /tmp/foo" "cat /tmp/foo"
```
Otherwise the shell will think that '-l' is just another command line argument being passed to sshpt.py and this has the potential to cause serious problems (e.g. running 'rm -rf' on your local machine by accident)