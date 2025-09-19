Based on the provided sources, here is a summary of the most important Linux commands, categorized for clarity:

### Refresher of Linux Commands

The sources offer a comprehensive list of commands for navigating the file system, managing files, processes, and network connections, as well as handling users and permissions.

#### File and Directory Management
These commands are essential for navigating and manipulating the file system:
*   **`pwd`**: Displays the present working directory.
*   **`ls`**: Lists directory contents.
    *   `ls -l`: Provides detailed information about files and directories.
    *   `ls -a`: Shows hidden files.
    *   `ls -la`: Combines detailed information with hidden files.
*   **`cd`**: Changes the current directory.
    *   `cd --`: Moves up one level in the directory hierarchy.
    *   `cd ~`: Navigates to the home directory.
*   **`mkdir`**: Creates new directories.
    *   `mkdir -p something/somethinginside`: Creates nested directories.
    *   `mkdir folder1 folder2`: Creates multiple directories.
*   **`rmdir folder`**: Removes an empty directory.
*   **`touch file.txt`**: Creates a new empty file or updates the timestamp of an existing one.
    *   `touch file1.txt file2.txt`: Creates multiple files.
*   **`cp`**: Copies files and directories.
    *   `cp file.txt anotherfile.txt`: Copies a file.
    *   `cp -r folder1 folder2`: Copies a folder and its contents recursively.
    *   `cp *.txt backup/`: Copies all `.txt` files to a specified directory.
*   **`mv`**: Moves or renames files and directories.
    *   `mv file.txt Documents/`: Moves a file.
    *   `mv file.txt newname.txt`: Renames a file.
    *   `mv folder1 folder2`: Renames a folder.
*   **`rm`**: Removes files and directories.
    *   `rm file.txt`: Removes a file.
    *   `rm -r folder`: Removes a folder and its contents recursively.
    *   `rm -f file.txt`: Forces removal without confirmation.
    *   `rm *.txt`: Deletes all files with the `.txt` extension.
*   **`find`**: Searches for files and directories based on various criteria.
    *   `find . -name "filename"`: Finds a file by name in the current directory.
    *   `find /home -name "*.txt"`: Finds all `.txt` files in the `/home` directory.
    *   `find -type d -name temp`: Finds directories named `temp`.
    *   `find . -size +100M`: Finds files larger than 100MB.
*   **`locate filename.txt`**: A faster way to find files, using a pre-built database.
    *   `locate "*pdf"`: Finds all files ending with `.pdf`.
*   **`ln`**: Creates links between files.
    *   `ln sourcefile link_name`: Creates a hard link.
    *   `ln -s file.txt softlink.txt`: Creates a symbolic link (shortcut).
    *   `ln -s "/pathtofolder" linkfolder`: Creates a symbolic link to a folder.

#### Text Processing and Viewing
These commands are used for displaying, searching, and manipulating text content in files:
*   **`cat file.txt`**: Displays the entire content of a file at once.
    *   `cat file1.txt file2.txt`: Displays contents of multiple files.
*   **`less file1.txt`**: Displays file content one screen at a time, allowing scrolling.
*   **`head file.txt`**: Shows the first 10 lines of a file.
    *   `head -n 5 file.txt`: Shows the first 5 lines.
*   **`tail file.txt`**: Shows the last 10 lines of a file.
    *   `tail -n 5 file.txt`: Shows the last 5 lines.
    *   `tail -f log.txt`**: Follows file changes in real-time, useful for monitoring logs.
*   **`grep`**: Searches for patterns within files.
    *   `grep "hello" file.txt`: Finds lines containing "hello".
    *   `grep -i "HELLO" file.txt`: Performs a case-insensitive search.
    *   `grep -r "error" /var/log`: Searches recursively in directories.
    *   `grep -v "pattern" file.txt`: Shows lines NOT matching the pattern.
    *   `grep -C "pattern" file.txt`: Shows lines before and after a match.
    *   `grep -rli "pattern" .`: Finds files containing a specific pattern.
*   **`echo "hi"`**: Prints text to the terminal.
    *   `echo "hi" > file.txt`: Overwrites file with text.
    *   `echo "hi" >> file.txt`: Appends text to a file.
*   **`wc file.txt`**: Counts lines, words, and characters in a file.
    *   `wc -l file.txt`: Counts lines.
    *   `wc -w file.txt`: Counts words.
    *   `wc -c file.txt`: Counts characters.
*   **`sort file.txt`**: Sorts lines of a file alphabetically.
    *   `sort -n file.txt`: Sorts numerically.
    *   `sort -r file.txt`: Sorts in reverse order.
    *   `sort -u file.txt`: Sorts and removes duplicates.

#### Archiving and Compression
These commands are used for bundling and compressing files:
*   **`tar`**: Archives files.
    *   `tar -cvf archive.tar files/`: Creates an archive.
    *   `tar -xvf archive.tar`: Extracts an archive.
    *   `tar -tvf archive.tar`: Lists archive contents.
    *   `tar -czvf archive.tar.gz files/`: Creates a compressed gzip archive.
    *   `tar -xzvf archive.tar.gz`: Extracts a compressed gzip archive.
*   **`gzip file.txt`**: Compresses a file using gzip.
    *   `gzip -d file.txt.gz`: Decompresses a gzip file.
*   **`zip -r archive.zip folder/`**: Creates a zip archive of a folder.
    *   `zip -d archive.zip file.txt`: Deletes a file from a zip archive.
*   **`unzip archive.zip`**: Extracts files from a zip archive.
    *   `unzip -l archive.zip`: Lists contents of a zip archive.
    *   `unzip -d /path/to/extract archive.zip`: Extracts to a specific directory.

#### Process Management
Commands for monitoring and controlling running processes:
*   **`ps`**: Displays currently running processes.
    *   `ps aux`: Shows all processes on the system.
    *   `ps aux | grep "firefox"`: Finds specific processes (e.g., Firefox).
    *   `ps -e -o pcpu,cpu,nice,state,cputime,args --sort pcpu | head -n 10`: Lists top CPU processes.
*   **`top`**: Provides a dynamic, real-time view of system processes.
    *   `top -u username`: Monitors processes for a specific user.
*   **`kill PID`**: Sends a signal to a process to terminate it (e.g., `kill 1234`).
    *   `kill -9 PID`: Forces termination of a process.
*   **`killall firefox`**: Kills all processes with a specific name (e.g., all Firefox processes).
*   **`which command`**: Shows the full path to an executable command (e.g., `which ls` returns `/usr/bin/ls`).

#### System Information and Disk Usage
Commands to check system resources and information:
*   **`df -h`**: Reports disk space usage in a human-readable format.
    *   `df -hT`: Includes filesystem type.
    *   `df -i`: Includes inode usage.
*   **`du -h file.txt`**: Estimates file space usage.
    *   `du -h folder/`: Estimates folder space usage.
    *   `du -sh folder/`: Shows only the total size of a folder.
    *   `du -h -d 1`: Shows the size of directories in the current location.
*   **`free -h`**: Displays amount of free and used memory in a human-readable format.
    *   `free -m`: Displays memory in MB.
*   **`uname`**: Prints system information.
    *   `uname -a`: Prints all system information.
    *   `uname -r`: Prints the kernel version.
*   **`date`**: Displays the current date and time.
*   **`cal`**: Displays a calendar.
*   **`history`**: Shows the history of commands executed by the user.
*   **`whoami`**: Displays the current user's username.

#### Networking
Commands for network diagnostics and remote access:
*   **`ping google.com`**: Tests network connectivity.
    *   `ping -c 4 google.com`: Sends only 4 packets.
*   **`wget`**: Downloads files from the internet.
    *   `wget https://example.com/file.zip`: Downloads a file.
    *   `wget -c https://example.com/file.zip`: Resumes an interrupted download.
*   **`curl`**: Transfers data from or to a server.
    *   `curl https://example.com`: Downloads a webpage.
    *   `curl -O https://example.com/file.zip`: Saves a file with its original name.
    *   `curl -X POST https://api.example.com`: Makes a POST request.
*   **`ifconfig`**: Displays and configures network interfaces.
    *   `ifconfig eth0`: Shows specific interface details.
    *   `ifconfig eth0 up`: Brings an interface up.
    *   `ifconfig eth0 down`: Brings an interface down.
*   **`netstat`**: Displays network connections, routing tables, interface statistics, etc.
    *   `netstat -a`: Shows all connections.
    *   `netstat -l`: Shows listening ports.
    *   `netstat -r`: Shows the routing table.
*   **`lsof`**: Lists open files.
    *   `lsof -i`: Lists open internet connections.
    *   `lsof -i :80`: Lists connections on port 80.
*   **`ssh`**: Secure Shell for remote access.
    *   `ssh username@hostname`: Connects to a remote host.
    *   `ssh -p 2222 username@hostname`: Connects on a specific port.
*   **`scp`**: Secure Copy for transferring files over SSH.
    *   `scp file.txt user@host:/path/`: Copies a file to a remote host.
    *   `scp user@host:/path/file.txt .`: Copies a file from a remote host.
    *   `scp -r folder/ user@host:/path/`: Copies a folder recursively.
*   **`ssh-keygen`**: Generates SSH key pairs.
    *   `ssh-keygen -t rsa`: Generates an RSA key pair.
    *   `ssh-keygen -t ed25519`: Generates an ED25519 key pair.
*   **`whois example.com`**: Retrieves domain information.
*   **`nc -l 8080`**: Listens on a specified port (Netcat).

#### User Management and Permissions
Commands for managing users, groups, and file permissions:
*   **`who`**: Shows users currently logged in.
    *   `who -H`: Shows header information.
*   **`id`**: Displays user and group information.
    *   `id username`: Shows information for a specific user.
*   **`groups`**: Lists groups a user belongs to.
    *   `groups username`: Lists groups for a specific user.
*   **`su username`**: Switches to another user.
    *   `su -`: Switches to the root user.
    *   `su - username`: Switches to a user with their environment.
*   **`sudo`**: Executes a command with root privileges.
*   **`chmod`**: Changes file permissions.
    *   `chmod 755 file.txt`: Sets specific permissions (read/write/execute for owner, read/execute for group and others).
    *   `chmod u+x file.txt`: Adds execute permission for the owner.
    *   `chmod g-w file.txt`: Removes write permission for the group.
    *   `chmod o+r file.txt`: Adds read permission for others.
    *   `chmod +x script.sh`: Makes a file executable for all.
*   **`chown`**: Changes file ownership.
*   **`chgrp`**: Changes group ownership of a file.
*   **`umask`**: Sets default file permissions for newly created files.
*   **`getfacl file.txt`**: Gets file Access Control Lists (ACLs).
*   **`setfacl`**: Sets file Access Control Lists (ACLs).
*   **`useradd`**: Adds a new user account.
*   **`adduser`**: Adds a new user interactively.
*   **`passwd`**: Changes a user's password.
*   **`deluser`**: Deletes a user account.
*   **`usermod`**: Modifies a user account.

#### System Control
Commands for shutting down or restarting the system:
*   **`shutdown -h now`**: Shuts down the system immediately.
*   **`shutdown -r now`**: Restarts the system immediately.
*   `shutdown -h +10`: Shuts down the system in 10 minutes.
*   **`reboot`**: Reboots the system.

#### Aliases and Timing
*   **`alias ll='ls -la'`**: Creates a custom shortcut or alias.
*   **`alias`**: Lists all current aliases.
*   **`unalias ll`**: Removes an alias.
*   **`time ls -la`**: Measures the execution time of a command.
