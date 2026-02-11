# Essential Linux Commands

## 📋 TẦM QUAN TRỌNG

> "Một DevOps Engineer giỏi phải thành thạo command line như ngôn ngữ mẹ đẻ"

80% công việc DevOps được thực hiện qua terminal/CLI.

## 🗂️ FILE & DIRECTORY OPERATIONS

### **Navigation**

```bash
# pwd - Print Working Directory
pwd
# /home/vnpt/projects

# cd - Change Directory
cd /var/log              # Absolute path
cd logs                  # Relative path
cd ~                     # Home directory
cd                       # Home directory (shortcut)
cd -                     # Previous directory
cd ..                    # Parent directory
cd ../..                 # Two levels up

# Pro tip: Use Tab for autocomplete
cd /var/l<TAB>          # Completes to /var/log
```

### **Listing Files**

```bash
# ls - List files
ls                       # Basic list
ls -l                    # Long format
ls -a                    # Show hidden files
ls -lh                   # Human readable sizes
ls -ltr                  # Sort by time, reverse
ls -lS                   # Sort by size
ls -R                    # Recursive
ls -d */                 # List directories only

# Practical examples
ls -lah                  # Most commonly used
ls -lt | head           # Latest 10 files
ls -lS | head           # Largest 10 files
ls *.log                # Filter by extension
ls -l | grep "Feb"      # Files from February

# tree - Directory tree (install: sudo apt install tree)
tree
tree -L 2               # Depth 2
tree -d                 # Directories only
tree -h                 # Human readable sizes
```

### **Creating & Deleting**

```bash
# touch - Create empty file or update timestamp
touch newfile.txt
touch file1.txt file2.txt file3.txt

# mkdir - Make directory
mkdir new_dir
mkdir -p parent/child/grandchild    # Create parent dirs
mkdir -p dir{1,2,3}                 # Create multiple dirs

# rm - Remove
rm file.txt                # Remove file
rm -i file.txt            # Interactive (confirm)
rm -f file.txt            # Force (no confirmation)
rm -r directory/          # Recursive (directories)
rm -rf directory/         # Force recursive (DANGEROUS!)

# Practical examples
rm *.log                  # Remove all .log files
rm -rf /tmp/old_data     # Remove directory with content
find . -name "*.tmp" -delete   # Remove all .tmp files recursively

# rmdir - Remove empty directory
rmdir empty_dir

# Pro tip: Use trash-cli instead of rm for safety
# Install: sudo apt install trash-cli
trash file.txt           # Move to trash instead of delete
```

### **Copying & Moving**

```bash
# cp - Copy
cp source.txt destination.txt       # Copy file
cp source.txt /backup/             # Copy to directory
cp -r directory/ /backup/          # Copy directory
cp -p file.txt backup.txt          # Preserve permissions
cp -u source.txt dest.txt          # Copy only if newer
cp -v source.txt dest.txt          # Verbose

# Practical examples
cp -r /var/www/html /backup/www_$(date +%Y%m%d)
cp /etc/nginx/nginx.conf{,.backup}  # Creates nginx.conf.backup

# mv - Move/Rename
mv old_name.txt new_name.txt       # Rename
mv file.txt /another/location/     # Move
mv -i source dest                  # Interactive
mv -v source dest                  # Verbose

# Practical examples
mv *.log /var/log/archive/         # Move all logs
rename 's/.txt/.md/' *.txt         # Rename extension (bulk)
```

## 📄 FILE VIEWING & EDITING

### **Viewing File Content**

```bash
# cat - Concatenate and display
cat file.txt
cat file1.txt file2.txt            # Multiple files
cat -n file.txt                    # Show line numbers
cat file.txt | less                # Pipe to less for pagination

# less - Page through file (recommended for large files)
less /var/log/syslog
# Navigation in less:
#   Space/f - Forward one page
#   b - Back one page
#   / - Search forward
#   ? - Search backward
#   n - Next search result
#   G - Go to end
#   g - Go to beginning
#   q - Quit

# head - First lines
head file.txt                      # First 10 lines
head -n 20 file.txt               # First 20 lines
head -n -5 file.txt               # All except last 5

# tail - Last lines
tail file.txt                      # Last 10 lines
tail -n 50 file.txt               # Last 50 lines
tail -f /var/log/syslog           # Follow (real-time)
tail -f -n 100 app.log            # Follow last 100 lines

# Practical examples
tail -f /var/log/nginx/access.log  # Monitor web traffic
tail -f /var/log/syslog | grep ERROR  # Filter errors
multitail file1.log file2.log      # Multiple files (install first)
```

### **Text Viewing & Editing**

```bash
# nano - Simple editor (beginner-friendly)
nano file.txt
# Ctrl+O: Save
# Ctrl+X: Exit
# Ctrl+K: Cut line
# Ctrl+U: Paste

# vi/vim - Powerful editor
vi file.txt
# Two modes: Command mode and Insert mode
# i: Insert mode
# Esc: Command mode
# :w - Save
# :q - Quit
# :wq - Save and quit
# :q! - Quit without saving
# dd - Delete line
# yy - Copy line
# p - Paste

# Practical vim tricks
vim +10 file.txt          # Open at line 10
vim -d file1 file2        # Diff mode
:set nu                   # Show line numbers
:%s/old/new/g            # Replace all
```

## SEARCH & FIND

### **grep - Search in Files**

```bash
# Basic grep
grep "pattern" file.txt
grep "error" /var/log/syslog

# Options
grep -i "error" file.txt         # Case insensitive
grep -r "TODO" /project          # Recursive
grep -n "error" file.txt         # Show line numbers
grep -v "error" file.txt         # Invert (exclude)
grep -c "error" file.txt         # Count matches
grep -l "error" *.log            # Files with match
grep -w "error" file.txt         # Whole word only
grep -A 5 "error" file.txt       # 5 lines after
grep -B 5 "error" file.txt       # 5 lines before
grep -C 5 "error" file.txt       # 5 lines context

# Practical examples
grep -r "TODO" . --include="*.py"    # Python files only
grep -i "error\|warning" *.log       # Multiple patterns
grep -rn "function myFunc" .         # Find function definition
ps aux | grep nginx                  # Find process
dmesg | grep -i error               # System errors

# Extended regex
grep -E "error|warning|critical" file.txt
egrep "(192\.168\.|10\.)" access.log  # IP addresses
```

### **find - Find Files**

```bash
# Basic find
find /path -name "filename"
find . -name "*.log"

# By type
find . -type f                   # Files only
find . -type d                   # Directories only
find . -type l                   # Symbolic links

# By size
find . -size +100M               # Larger than 100MB
find . -size -1k                 # Smaller than 1KB
find . -size 50M                 # Exactly 50MB

# By time
find . -mtime -7                 # Modified in last 7 days
find . -mtime +30                # Modified more than 30 days ago
find . -mmin -60                 # Modified in last 60 minutes
find . -atime -1                 # Accessed in last 1 day

# By permissions
find . -perm 777                 # Exact permission
find . -perm -u+x                # User executable

# By owner
find . -user vnpt
find . -group developers

# Actions
find . -name "*.tmp" -delete     # Delete found files
find . -name "*.log" -exec gzip {} \;   # Compress logs
find . -type f -exec chmod 644 {} \;    # Change permissions

# Complex examples
# Find large log files older than 30 days
find /var/log -name "*.log" -size +100M -mtime +30

# Find and delete empty files
find . -type f -empty -delete

# Find recently modified config files
find /etc -name "*.conf" -mtime -1
```

### **locate - Quick Search**

```bash
# Faster than find (uses pre-built database)
locate nginx.conf
locate -i pattern                # Case insensitive
locate -c pattern                # Count only

# Update database
sudo updatedb

# Limit results
locate -n 10 ".conf"            # First 10 results
```

## FILE ANALYSIS

### **wc - Word Count**

```bash
wc file.txt                      # Lines, words, bytes
wc -l file.txt                   # Lines only
wc -w file.txt                   # Words only
wc -c file.txt                   # Bytes
wc -m file.txt                   # Characters

# Practical examples
cat access.log | wc -l           # Count log entries
find . -name "*.py" | wc -l      # Count Python files
```

### **sort & uniq**

```bash
# sort - Sort lines
sort file.txt
sort -r file.txt                 # Reverse
sort -n file.txt                 # Numeric sort
sort -k 2 file.txt               # Sort by column 2
sort -u file.txt                 # Unique lines
sort -t: -k3 -n /etc/passwd      # Sort passwd by UID

# uniq - Remove duplicates (requires sorted input)
sort file.txt | uniq
sort file.txt | uniq -c          # Count occurrences
sort file.txt | uniq -d          # Only duplicates
sort file.txt | uniq -u          # Only unique lines

# Practical examples
cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -rn
# IP addresses sorted by frequency
```

### **cut & awk**

```bash
# cut - Extract columns
cut -d: -f1 /etc/passwd          # First field (delimiter :)
cut -c1-10 file.txt              # Characters 1-10
cut -d, -f2,4 data.csv           # CSV columns 2 and 4

# awk - Pattern scanning and processing
awk '{print $1}' file.txt        # First column
awk '{print $1, $3}' file.txt    # Columns 1 and 3
awk -F: '{print $1}' /etc/passwd # Custom delimiter
awk '$3 > 100' data.txt          # Filter rows

# Practical awk examples
# Sum column
awk '{sum += $1} END {print sum}' numbers.txt

# Average
awk '{sum += $1; count++} END {print sum/count}' numbers.txt

# Format output
awk '{printf "%-20s %s\n", $1, $2}' data.txt

# Analyze access log
awk '{print $1}' access.log | sort| uniq -c | sort -rn | head -10
# Top 10 IP addresses
```

### **sed - Stream Editor**

```bash
# Basic substitution
sed 's/old/new/' file.txt        # First occurrence per line
sed 's/old/new/g' file.txt       # All occurrences
sed 's/old/new/gi' file.txt      # Case insensitive

# In-place editing
sed -i 's/old/new/g' file.txt    # Modify file directly
sed -i.bak 's/old/new/g' file.txt # Create backup

# Delete lines
sed '5d' file.txt                # Delete line 5
sed '1,10d' file.txt             # Delete lines 1-10
sed '/pattern/d' file.txt        # Delete matching lines

# Print lines
sed -n '10,20p' file.txt         # Print lines 10-20
sed -n '/ERROR/p' file.txt       # Print matching lines

# Practical examples
# Remove comments and blank lines
sed '/^#/d' /etc/nginx/nginx.conf | sed '/^$/d'

# Replace multiple patterns
sed -e 's/foo/bar/g' -e 's/hello/world/g' file.txt

# Add line numbers
sed = file.txt | sed 'N;s/\n/\t/'
```

## 🔧 SYSTEM COMMANDS

### **Process Management**

```bash
# ps - Process status
ps                               # Current shell processes
ps aux                          # All processes
ps aux | grep nginx             # Find specific process
ps -ef                          # Full format
ps -u vnpt                      # User processes
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head

# top - Real-time process viewer
top
# Interactive commands:
#   k - Kill process
#   r - Renice
#   q - Quit
#   M - Sort by memory
#   P - Sort by CPU
#   1 - Show all CPUs

# htop - Better top (install: sudo apt install htop)
htop
# F9 - Kill, F6 - Sort, F4 - Filter

# kill - Terminate process
kill PID                        # SIGTERM (graceful)
kill -9 PID                     # SIGKILL (force)
kill -15 PID                    # SIGTERM (explicit)
killall nginx                   # Kill by name
pkill nginx                     # Kill by pattern

# Practical examples
# Kill all Python processes
pkill -9 python

# Find and kill
ps aux | grep myapp | awk '{print $2}' | xargs kill
```

### **System Resources**

```bash
# df - Disk space
df -h                           # Human readable
df -i                           # Inodes
df -T                           # File system type

# du - Disk usage
du -sh /var/log                 # Summary
du -h --max-depth=1 /var        # One level deep
du -sh * | sort -rh             # Sort by size

# free - Memory usage
free -h                         # Human readable
free -m                         # In MB
free -s 5                       # Update every 5 seconds

# uptime - System uptime and load
uptime

# uname - System information
uname -a                        # All info
uname -r                        # Kernel version
```

## NETWORK COMMANDS

```bash
# ping - Test connectivity
ping google.com
ping -c 4 8.8.8.8              # 4 packets only

# curl - Transfer data
curl https://api.github.com
curl -I https://google.com      # Headers only
curl -o file.html https://example.com  # Save to file
curl -X POST -d "data" url      # POST request

# wget - Download files
wget https://example.com/file.zip
wget -c url                     # Resume download
wget -r url                     # Recursive download

# netstat - Network statistics
netstat -tuln                   # Listening ports
netstat -anp | grep :80         # Port 80 connections

# ss - Socket statistics (modern netstat)
ss -tuln                        # Listening TCP/UDP
ss -s                          # Summary

# ip - Network config
ip addr show                    # IP addresses
ip link show                    # Network interfaces
ip route show                   # Routing table
```

## Cheat Sheet

```bash
# Most commonly used commands
ls -lah          # List files
cd /path         # Change directory
pwd              # Current directory
cat file         # View file
tail -f log      # Follow log
grep pattern     # Search
find . -name     # Find files
ps aux           # Processes
df -h            # Disk space
free -h          # Memory
top              # Resource monitor
```

---

**Tiếp theo**: [Shell Scripting →](../shell-scripting/bash-basics.md)
