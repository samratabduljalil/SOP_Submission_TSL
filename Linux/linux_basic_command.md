# Linux Basic Commands 

---

## `ls` — List files and directories

```bash
ls                  # List files in current directory
ls -a               # Show hidden files (dotfiles)
ls -l               # Long format — permissions, owner, size, date
ls -h               # Human-readable sizes (KB, MB, GB)
ls -t               # Sort by modification time, newest first

```

---

## `su` — Switch User

```bash
su                          # Switch to root user
su username                 # Switch to a specific user
su - username               # Switch with full login shell
su -c "command" username    # Run single command as another user
```

---

## `cd` — Change Directory

```bash
cd /path/to/dir         # Go to absolute path
cd ..                   # Go up one level
cd ~                    # Go to home directory
cd -                    # Go to previous directory
cd /                    # Go to root directory
```

---

## `pwd` — Print Working Directory

```bash
pwd         # Show current directory path

```

---

## `systemctl` — Manage System Services

```bash
systemctl start pgpool          # Start a service
systemctl stop pgpool           # Stop a service
systemctl restart pgpool        # Restart a service
systemctl reload pgpool         # Reload config without full restart
systemctl status pgpool         # Show current status of a service
systemctl enable pgpool         # Enable service to start on boot
systemctl disable pgpool        # Disable service from starting on boot
systemctl daemon-reload         # Reload systemd after config changes
```

---

## `history` — Command History

```bash
history             # Show all command history
history 20          # Show last 20 commands
history -c          # Clear all history
history -d 42       # Delete history entry number 42
```

---

## `cat` — Display File Contents

```bash
cat file.txt                    # Display file contents
cat -n file.txt                 # Show with line numbers
cat file1 file2 > merged.txt    # Merge two files into one
```

---

## `top` — Real-Time Process Monitor

```bash
top                 # Launch live process monitor
top -u username     # Show only processes of a specific user
top -p 1234         # Monitor one specific process by PID
top -n 1            # Run once and exit (non-interactive)
top -d 5            # Set refresh interval to 5 seconds
```

---

## `htop` — Enhanced Process Viewer

```bash
htop                    # Launch htop interactive viewer
htop -u username        # Filter processes by user
htop -p 1234,5678       # Monitor specific PIDs
htop -d 10              # Set refresh delay (tenths of seconds)
```

---

## `mv` — Move or Rename Files

```bash
mv file.txt /dest/          # Move file to destination
mv old.txt new.txt          # Rename a file
mv -i file.txt /dest/       # Prompt before overwriting
mv -v file.txt /dest/       # Verbose — show what is being moved
mv dir/ /new/path/          # Move entire directory
```

---

## `cp` — Copy Files or Directories

```bash
cp file.txt /dest/          # Copy file to destination
cp -r dir/ /dest/           # Copy directory recursively
cp -p file.txt /dest/       # Preserve permissions and timestamps
cp -i file.txt /dest/       # Prompt before overwriting
cp -v file.txt /dest/       # Verbose — show files being copied
```

---

## `rm` — Remove Files or Directories

```bash
rm file.txt         # Delete a file
rm -r dir/          # Delete directory and its contents recursively
rm -f file.txt      # Force delete without prompt
rm -rf dir/         # Force delete directory recursively 
rm -i file.txt      # Prompt before every deletion
rm -v file.txt      # Verbose — show what is being deleted
rm *.log            # Delete all files matching a pattern
```

---

## `chown` — Change File Owner

```bash
chown user file                 # Change owner of a file
chown user:group file           # Change both owner and group
chown -R user:group dir/        # Recursively change ownership
```

---

## `chmod` — Change File Permissions

```bash
chmod 777 file          # rwxrwxrwx (everyone full access)
chmod g+x file           # Add execute permission for group
chmod o-x file           # Remove execute permission from others
chmod u+w file          # Add write permission for owner only
chmod -R 755 dir/       # Apply permissions recursively
chmod a+r file          # Add read permission for all users
```

---

## `less` — Scroll Through File Contents

```bash
less file.txt           # Open file for scrolling
less -N file.txt        # Show line numbers
```

---

## `more` — Page Through File Contents

```bash
more file.txt           # Page through a file
more -n 20 file.txt     # Show 20 lines per page
```

---

## `tail` — View End of File

```bash
tail file.txt           # Show last 10 lines (default)
tail -n 50 file.txt     # Show last 50 lines
tail -f file.log        # Follow and show live updates in real time
```

---

## `alias` — Create Command Shortcuts

```bash
alias                               # List all current aliases
alias ll='ls -alh'                  # Create a new alias
unalias ll                          # Remove a specific alias
unalias -a                          # Remove all aliases
```

---

## `wc` — Word & Line Count

```bash
wc file.txt             # Show lines, words, and bytes together
wc -l file.txt          # Count lines only
wc -w file.txt          # Count words only
wc -c file.txt          # Count bytes only
```

---

## `sort` — Sort Lines of Text

```bash
sort file.txt               # Sort alphabetically A to Z
sort -r file.txt            # Reverse sort Z to A
sort -n file.txt            # Sort numerically
sort -u file.txt            # Sort and remove duplicates
```

---

## `cmp` — Compare Two Files

```bash
cmp file1 file2             # Compare and show first difference
```
---

## `tar` — Archive Files

```bash
tar -cvf archive.tar dir/           # Create archive
tar -xvf archive.tar                # Extract archive
tar -czvf archive.tar.gz dir/       # Create gzip-compressed archive
tar -xzvf archive.tar.gz            # Extract gzip archive
tar -tvf archive.tar                # List contents without extracting
tar -xvf archive.tar -C /dest/      # Extract to a specific directory
```

---

## `gzip` — Compress Files

```bash
gzip file.txt           # Compress file (replaces original)
gzip -k file.txt        # Compress and keep the original file
gzip -d file.txt.gz     # Decompress a .gz file
```

---

## `zip` — Create ZIP Archives

```bash
zip archive.zip file1 file2     # Zip specific files
zip -r archive.zip dir/         # Zip a directory recursively
```

---

## `unzip` — Extract ZIP Archives

```bash
unzip archive.zip               # Extract to current directory
unzip archive.zip -d /dest/     # Extract to a specific directory
unzip -l archive.zip            # List contents without extracting
```

---

## `mkdir` — Create Directories

```bash
mkdir dirname               # Create a single directory
mkdir -p a/b/c              # Create nested directories 
mkdir dir1 dir2 dir3        # Create multiple directories at once
```

---

## `rmdir` — Remove Empty Directories

```bash
rmdir dirname           # Remove an empty directory
rmdir -p a/b/c          # Remove nested empty directories
rmdir -v dirname        # Verbose — show directory being removed
rmdir dir1 dir2         # Remove multiple empty directories
```

---

## `chgrp` — Change Group Ownership

```bash
chgrp groupname file                # Change group of a file
chgrp -R groupname dir/             # Recursively change group ownership
chgrp -v groupname file             # Verbose — show changes made
```

---

## `watch` — Repeat a Command Live

```bash
watch command                           # Run command every 2 seconds (default)
watch -n 5 command                      # Run every 5 seconds

```

---

## `uptime` — System Uptime

```bash
uptime          # Show uptime, logged-in users, and load averages
uptime -p       # Show uptime in human-readable format
uptime -s       # Show exact date and time system was booted
```

---

## `grep` — Search Text Patterns

```bash
grep "pattern" file.txt             # Basic pattern search
grep -i "pattern" file.txt          # Case-insensitive search
grep -r "pattern" dir/              # Recursive search in a directory
grep -n "pattern" file.txt          # Show matching line numbers
grep -v "pattern" file.txt          # Invert — show non-matching lines
grep -c "pattern" file.txt          # Count matching lines
grep -l "pattern" *.txt             # List only filenames with matches
grep -w "word" file.txt             # Match whole word only
```

---

## `sed` — Stream Editor

```bash
sed 's/old/new/' file.txt           # Replace first occurrence per line
sed 's/old/new/g' file.txt          # Replace all occurrences per line
sed -i 's/old/new/g' file.txt       # Edit file in-place (modifies original)
sed -n '5,10p' file.txt             # Print only lines 5 to 10
sed '/pattern/d' file.txt           # Delete lines matching pattern
sed -n '/pattern/p' file.txt        # Print only lines matching pattern
sed '3d' file.txt                   # Delete line number 3
```

---

## `command &` — Background Jobs

```bash
nohup ./script.sh &                 # Background, survives terminal close
```

---

## `jobs` — List Background Jobs

```bash
jobs            # List all jobs
jobs -l         # List jobs with their PIDs
```

---

## `bg` — Resume Job in Background

```bash
bg          # Resume last stopped job in background

```

---

## `fg` — Bring Job to Foreground

```bash
fg          # Bring last job to foreground

```

---

## `nohup` — Run Immune to Hangup

```bash
nohup command &                         # Run immune to hangup signal
nohup ./script.sh > output.log &        # Save output to a custom log file
```

---

## `nice` — Set Process Priority

```bash
nice command                # Run with default niceness (+10)
nice -n 10 command          # Lower priority (less CPU usage)
nice -n -10 command         # Higher priority (needs root)
nice -n 19 command          # Lowest possible priority
nice -n -20 command         # Highest possible priority (root only)
```

---

## `renice` — Change Running Process Priority

```bash
renice -n 5 -p 1234             # Set niceness of a process by PID
renice -n -5 1234               # Increase process priority (needs root)
renice -n 10 -u username        # Renice all processes of a user
renice -n 5 -g groupname        # Renice all processes of a group
```

---

## `journalctl` — View System Logs

```bash
journalctl                                          # Show all logs
journalctl -u pgpool                                # Logs for a specific service
journalctl -f                                       # Follow live log output
journalctl -n 50                                    # Show last 50 log entries
journalctl -xe                                      # Recent logs with explanations
journalctl -u pgpool -f                             # Follow a specific service live
```

---

## `touch` — Create File or Update Timestamp

```bash
touch file.txt                  # Create empty file or update timestamp
touch file1 file2 file3         # Create multiple files at once
```


