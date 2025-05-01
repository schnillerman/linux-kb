# Facilitations
## Elevated Rights in Shell
### Add user to sudoers
Add your main user to sudoers and possibly (your own risk) remove the password authentication (every 15 minutes) for sudo commands:
1. `sudo adduser <username> sudo`
2. In `/etc/sudoers`, replace line `%sudo   ALL=(ALL:ALL) ALL` with `%sudo   ALL=(ALL:ALL) NOPASSWD: ALL`
# Problems
## Lock Out
### Sudoer Syntax Errors
#### Problem
You are as stupid as me and lock yourself out of being able to do anything in the shell and get a similar error message as the following:
```
<user>@<host>:~$ sudo su
/etc/sudoers:50:25: Syntax error
%sudo   ALL=(ALL:ALL) ALL NOPASSWD: ALL
                        ^~~~~~~~~
[sudo] password for <user>:
Sorry, user <user> is not allowed to execute »/usr/bin/su« as root on <host>.
```
#### Solution
The following is a working solution as of 2025-05-01.
1. Open second SSH session to same host
2. 1st session: get bash PID
   ```
   echo $$
   ```
3. 2nd session: start auth agent
   ```
   pkttyagent --process <PID from 1st session>
   ```
4. 1st session:
   execute pkexec as sudo alternative, followed by command to gain rights to edit `/etc/suoders`
   ```
   pkexec {chown <user>:<user> /etc/sudoers,chmod 777 /etc/sudoers}
   ```
5. 2nd session: enter su password
6. 1st session
   1. correct `/etc/sudoers`
      ```
      nano /etc/sudoers
      ```
   2. restore sudoers ownership and permissions:
      ```
      pkexec {chown root:root /etc/sudoers,chmod 440 /etc/sudoers}
      ```
7. 2nd session: authenticate

This brilliant solution / hack is taken from [How to restore a broken sudoers file without being able to use sudo?](https://unix.stackexchange.com/questions/677591/how-to-restore-a-broken-sudoers-file-without-being-able-to-use-sudo)
