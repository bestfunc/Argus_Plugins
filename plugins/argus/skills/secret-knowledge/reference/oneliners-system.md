<!-- Vendored from https://github.com/trimstray/the-book-of-secret-knowledge (MIT License). Snapshot: 2026-06-14. 本文件是该知识库按分类拆分出来的一部分，供 Argus secret-knowledge skill 按需 Grep 检索。原始版权归 trimstray 及贡献者所有，保留 MIT 许可。 -->

#### Shell One-liners &nbsp;[<sup>[TOC]</sup>](#anger-table-of-contents)

##### Table of Contents

  * [terminal](#tool-terminal)
  * [busybox](#tool-busybox)
  * [mount](#tool-mount)
  * [fuser](#tool-fuser)
  * [lsof](#tool-lsof)
  * [ps](#tool-ps)
  * [top](#tool-top)
  * [vmstat](#tool-vmstat)
  * [iostat](#tool-iostat)
  * [strace](#tool-strace)
  * [kill](#tool-kill)
  * [find](#tool-find)
  * [diff](#tool-diff)
  * [vimdiff](#tool-vimdiff)
  * [tail](#tool-tail)
  * [cpulimit](#tool-cpulimit)
  * [pwdx](#tool-pwdx)
  * [tr](#tool-tr)
  * [chmod](#tool-chmod)
  * [who](#tool-who)
  * [last](#tool-last)
  * [screen](#tool-screen)
  * [script](#tool-script)
  * [du](#tool-du)
  * [inotifywait](#tool-inotifywait)
  * [openssl](#tool-openssl)
  * [secure-delete](#tool-secure-delete)
  * [dd](#tool-dd)
  * [gpg](#tool-gpg)
  * [system-other](#tool-system-other)
  * [curl](#tool-curl)
  * [httpie](#tool-httpie)
  * [ssh](#tool-ssh)
  * [linux-dev](#tool-linux-dev)
  * [tcpdump](#tool-tcpdump)
  * [tcpick](#tool-tcpick)
  * [ngrep](#tool-ngrep)
  * [hping3](#tool-hping3)
  * [nmap](#tool-nmap)
  * [netcat](#tool-netcat)
  * [socat](#tool-socat)
  * [p0f](#tool-p0f)
  * [gnutls-cli](#tool-gnutls-cli)
  * [netstat](#tool-netstat)
  * [rsync](#tool-rsync)
  * [host](#tool-host)
  * [dig](#tool-dig)
  * [certbot](#tool-certbot)
  * [network-other](#tool-network-other)
  * [git](#tool-git)
  * [awk](#tool-awk)
  * [sed](#tool-sed)
  * [grep](#tool-grep)
  * [perl](#tool-perl)

##### Tool: [terminal](https://en.wikipedia.org/wiki/Linux_console)

###### Reload shell without exit

```bash
exec $SHELL -l
```

###### Close shell keeping all subprocess running

```bash
disown -a && exit
```

###### Exit without saving shell history

```bash
kill -9 $$
unset HISTFILE && exit
```

###### Perform a branching conditional

```bash
true && echo success
false || echo failed
```

###### Pipe stdout and stderr to separate commands

```bash
some_command > >(/bin/cmd_for_stdout) 2> >(/bin/cmd_for_stderr)
```

###### Redirect stdout and stderr each to separate files and print both to the screen

```bash
(some_command 2>&1 1>&3 | tee errorlog ) 3>&1 1>&2 | tee stdoutlog
```

###### List of commands you use most often

```bash
history | \
awk '{CMD[$2]++;count++;}END { for (a in CMD)print CMD[a] " " CMD[a]/count*100 "% " a;}' | \
grep -v "./" | \
column -c3 -s " " -t | \
sort -nr | nl |  head -n 20
```

###### Sterilize bash history

```bash
function sterile() {

  history | awk '$2 != "history" { $1=""; print $0 }' | egrep -vi "\
curl\b+.*(-E|--cert)\b+.*\b*|\
curl\b+.*--pass\b+.*\b*|\
curl\b+.*(-U|--proxy-user).*:.*\b*|\
curl\b+.*(-u|--user).*:.*\b*
.*(-H|--header).*(token|auth.*)\b+.*|\
wget\b+.*--.*password\b+.*\b*|\
http.?://.+:.+@.*\
" > $HOME/histbuff; history -r $HOME/histbuff;

}

export PROMPT_COMMAND="sterile"
```

  > Look also: [A naive utility to censor credentials in command history](https://github.com/lbonanomi/go/blob/master/revisionist.go).

###### Quickly backup a file

```bash
cp filename{,.orig}
```

###### Empty a file (truncate to 0 size)

```bash
>filename
```

###### Delete all files in a folder that don't match a certain file extension

```bash
rm !(*.foo|*.bar|*.baz)
```

###### Pass multi-line string to a file

```bash
# cat  >filename ... - overwrite the file
# cat >>filename ... - append to a file
cat > filename << __EOF__
data data data
__EOF__
```

###### Edit a file on a remote host using vim

```bash
vim scp://user@host//etc/fstab
```

###### Create a directory and change into it at the same time

```bash
mkd() { mkdir -p "$@" && cd "$@"; }
```

###### Convert uppercase files to lowercase files

```bash
rename 'y/A-Z/a-z/' *
```

###### Print a row of characters across the terminal

```bash
printf "%`tput cols`s" | tr ' ' '#'
```

###### Show shell history without line numbers

```bash
history | cut -c 8-
fc -l -n 1 | sed 's/^\s*//'
```

###### Run command(s) after exit session

```bash
cat > /etc/profile << __EOF__
_after_logout() {

  username=$(whoami)

  for _pid in $(ps afx | grep sshd | grep "$username" | awk '{print $1}') ; do

    kill -9 $_pid

  done

}
trap _after_logout EXIT
__EOF__
```

###### Generate a sequence of numbers

```bash
for ((i=1; i<=10; i+=2)) ; do echo $i ; done
# alternative: seq 1 2 10

for ((i=5; i<=10; ++i)) ; do printf '%02d\n' $i ; done
# alternative: seq -w 5 10

for i in {1..10} ; do echo $i ; done
```

###### Simple Bash filewatching

```bash
unset MAIL; export MAILCHECK=1; export MAILPATH='$FILE_TO_WATCH?$MESSAGE'
```

---

##### Tool: [busybox](https://www.busybox.net/)

###### Static HTTP web server

```bash
busybox httpd -p $PORT -h $HOME [-c httpd.conf]
```

___

##### Tool: [mount](https://en.wikipedia.org/wiki/Mount_(Unix))

###### Mount a temporary ram partition

```bash
mount -t tmpfs tmpfs /mnt -o size=64M
```

  * `-t` - filesystem type
  * `-o` - mount options

###### Remount a filesystem as read/write

```bash
mount -o remount,rw /
```

___

##### Tool: [fuser](https://en.wikipedia.org/wiki/Fuser_(Unix))

###### Show which processes use the files/directories

```bash
fuser /var/log/daemon.log
fuser -v /home/supervisor
```

###### Kills a process that is locking a file

```bash
fuser -ki filename
```

  * `-i` - interactive option

###### Kills a process that is locking a file with specific signal

```bash
fuser -k -HUP filename
```

  * `--list-signals` - list available signal names

###### Show what PID is listening on specific port

```bash
fuser -v 53/udp
```

###### Show all processes using the named filesystems or block device

```bash
fuser -mv /var/www
```

___

##### Tool: [lsof](https://en.wikipedia.org/wiki/Lsof)

###### Show process that use internet connection at the moment

```bash
lsof -P -i -n
```

###### Show process that use specific port number

```bash
lsof -i tcp:443
```

###### Lists all listening ports together with the PID of the associated process

```bash
lsof -Pan -i tcp -i udp
```

###### List all open ports and their owning executables

```bash
lsof -i -P | grep -i "listen"
```

###### Show all open ports

```bash
lsof -Pnl -i
```

###### Show open ports (LISTEN)

```bash
lsof -Pni4 | grep LISTEN | column -t
```

###### List all files opened by a particular command

```bash
lsof -c "process"
```

###### View user activity per directory

```bash
lsof -u username -a +D /etc
```

###### Show 10 largest open files

```bash
lsof / | \
awk '{ if($7 > 1048576) print $7/1048576 "MB" " " $9 " " $1 }' | \
sort -n -u | tail | column -t
```

###### Show current working directory of a process

```bash
lsof -p <PID> | grep cwd
```

___

##### Tool: [ps](https://en.wikipedia.org/wiki/Ps_(Unix))

###### Show a 4-way scrollable process tree with full details

```bash
ps awwfux | less -S
```

###### Processes per user counter

```bash
ps hax -o user | sort | uniq -c | sort -r
```

###### Show all processes by name with main header

```bash
ps -lfC nginx
```

___

##### Tool: [find](https://en.wikipedia.org/wiki/Find_(Unix))

###### Find files that have been modified on your system in the past 60 minutes

```bash
find / -mmin 60 -type f
```

###### Find all files larger than 20M

```bash
find / -type f -size +20M
```

###### Find duplicate files (based on MD5 hash)

```bash
find -type f -exec md5sum '{}' ';' | sort | uniq --all-repeated=separate -w 33
```

###### Change permission only for files

```bash
cd /var/www/site && find . -type f -exec chmod 766 {} \;
cd /var/www/site && find . -type f -exec chmod 664 {} +
```

###### Change permission only for directories

```bash
cd /var/www/site && find . -type d -exec chmod g+x {} \;
cd /var/www/site && find . -type d -exec chmod g+rwx {} +
```

###### Find files and directories for specific user/group

```bash
# User:
find . -user <username> -print
find /etc -type f -user <username> -name "*.conf"

# Group:
find /opt -group <group>
find /etc -type f -group <group> -iname "*.conf"
```

###### Find files and directories for all without specific user/group

```bash
# User:
find . \! -user <username> -print

# Group:
find . \! -group <group>
```

###### Looking for files/directories that only have certain permission

```bash
# User
find . -user <username> -perm -u+rw # -rw-r--r--
find /home -user $(whoami) -perm 777 # -rwxrwxrwx

# Group:
find /home -type d -group <group> -perm 755 # -rwxr-xr-x
```

###### Delete older files than 60 days

```bash
find . -type f -mtime +60 -delete
```

###### Recursively remove all empty sub-directories from a directory

```bash
find . -depth  -type d  -empty -exec rmdir {} \;
```

###### How to find all hard links to a file

```bash
find </path/to/dir> -xdev -samefile filename
```

###### Recursively find the latest modified files

```bash
find . -type f -exec stat --format '%Y :%y %n' "{}" \; | sort -nr | cut -d: -f2- | head
```

###### Recursively find/replace of a string with sed

```bash
find . -not -path '*/\.git*' -type f -print0 | xargs -0 sed -i 's/foo/bar/g'
```

###### Recursively find/replace of a string in directories and file names

```bash
find . -depth -name '*test*' -execdir bash -c 'mv -v "$1" "${1//foo/bar}"' _ {} \;
```

###### Recursively find suid executables

```bash
find / \( -perm -4000 -o -perm -2000 \) -type f -exec ls -la {} \;
```

___

##### Tool: [top](https://en.wikipedia.org/wiki/Top_(software))

###### Use top to monitor only all processes with the specific string

```bash
top -p $(pgrep -d , <str>)
```

  * `<str>` - process containing string (eg. nginx, worker)

___

##### Tool: [vmstat](https://en.wikipedia.org/wiki/Vmstat)

###### Show current system utilization (fields in kilobytes)

```bash
vmstat 2 20 -t -w
```

  * `2` - number of times with a defined time interval (delay)
  * `20` - each execution of the command (count)
  * `-t` - show timestamp
  * `-w` - wide output
  * `-S M` - output of the fields in megabytes instead of kilobytes

###### Show current system utilization will get refreshed every 5 seconds

```bash
vmstat 5 -w
```

###### Display report a summary of disk operations

```bash
vmstat -D
```

###### Display report of event counters and memory stats

```bash
vmstat -s
```

###### Display report about kernel objects stored in slab layer cache

```bash
vmstat -m
```

##### Tool: [iostat](https://en.wikipedia.org/wiki/Iostat)

###### Show information about the CPU usage, and I/O statistics about all the partitions

```bash
iostat 2 10 -t -m
```

  * `2` - number of times with a defined time interval (delay)
  * `10` - each execution of the command (count)
  * `-t` - show timestamp
  * `-m` - fields in megabytes (`-k` - in kilobytes, default)

###### Show information only about the CPU utilization

```bash
iostat 2 10 -t -m -c
```

###### Show information only about the disk utilization

```bash
iostat 2 10 -t -m -d
```

###### Show information only about the LVM utilization

```bash
iostat -N
```

___

##### Tool: [strace](https://en.wikipedia.org/wiki/Strace)

###### Track with child processes

```bash
# 1)
strace -f -p $(pidof glusterfsd)

# 2)
strace -f $(pidof php-fpm | sed 's/\([0-9]*\)/\-p \1/g')
```

###### Track process with 30 seconds limit

```bash
timeout 30 strace $(< /var/run/zabbix/zabbix_agentd.pid)
```

###### Track processes and redirect output to a file

```bash
ps auxw | grep '[a]pache' | awk '{print " -p " $2}' | \
xargs strace -o /tmp/strace-apache-proc.out
```

###### Track with print time spent in each syscall and limit length of print strings

```bash
ps auxw | grep '[i]init_policy' | awk '{print " -p " $2}' | \
xargs strace -f -e trace=network -T -s 10000
```

###### Track the open request of a network port

```bash
strace -f -e trace=bind nc -l 80
```

###### Track the open request of a network port (show TCP/UDP)

```bash
strace -f -e trace=network nc -lu 80
```

___

##### Tool: [kill](https://en.wikipedia.org/wiki/Kill_(command))

###### Kill a process running on port

```bash
kill -9 $(lsof -i :<port> | awk '{l=$2} END {print l}')
```

___

##### Tool: [diff](https://en.wikipedia.org/wiki/Diff)

###### Compare two directory trees

```bash
diff <(cd directory1 && find | sort) <(cd directory2 && find | sort)
```

###### Compare output of two commands

```bash
diff <(cat /etc/passwd) <(cut -f2 /etc/passwd)
```

___

##### Tool: [vimdiff](http://vimdoc.sourceforge.net/htmldoc/diff.html)

###### Highlight the exact differences, based on characters and words

```bash
vimdiff file1 file2
```

###### Compare two JSON files

```bash
vimdiff <(jq -S . A.json) <(jq -S . B.json)
```

###### Compare Hex dump

```bash
d(){ vimdiff <(f $1) <(f $2);};f(){ hexdump -C $1 | cut -d' ' -f3- | tr -s ' ';}; d ~/bin1 ~/bin2
```

###### diffchar

Save [diffchar](https://raw.githubusercontent.com/vim-scripts/diffchar.vim/master/plugin/diffchar.vim) @ `~/.vim/plugins`

Click `F7` to switch between diff modes

Usefull `vimdiff` commands:

* `qa` to exit all windows
* `:vertical resize 70` to resize window
* set window width `Ctrl+W [N columns]+(Shift+)<\>`

___

##### Tool: [tail](https://en.wikipedia.org/wiki/Tail_(Unix))

###### Annotate tail -f with timestamps

```bash
tail -f file | while read ; do echo "$(date +%T.%N) $REPLY" ; done
```

###### Analyse an Apache access log for the most common IP addresses

```bash
tail -10000 access_log | awk '{print $1}' | sort | uniq -c | sort -n | tail
```

###### Analyse web server log and show only 5xx http codes

```bash
tail -n 100 -f /path/to/logfile | grep "HTTP/[1-2].[0-1]\" [5]"
```

___

##### Tool: [tar](https://en.wikipedia.org/wiki/Tar_(computing))

###### System backup with exclude specific directories

```bash
cd /
tar -czvpf /mnt/system$(date +%d%m%Y%s).tgz --directory=/ \
--exclude=proc/* --exclude=sys/* --exclude=dev/* --exclude=mnt/* .
```

###### System backup with exclude specific directories (pigz)

```bash
cd /
tar cvpf /backup/snapshot-$(date +%d%m%Y%s).tgz --directory=/ \
--exclude=proc/* --exclude=sys/* --exclude=dev/* \
--exclude=mnt/* --exclude=tmp/* --use-compress-program=pigz .
```

___

##### Tool: [dump](https://en.wikipedia.org/wiki/Dump_(program))

###### System backup to file

```bash
dump -y -u -f /backup/system$(date +%d%m%Y%s).lzo /
```

###### Restore system from lzo file

```bash
cd /
restore -rf /backup/system$(date +%d%m%Y%s).lzo
```

___

##### Tool: [cpulimit](http://cpulimit.sourceforge.net/)

###### Limit the cpu usage of a process

```bash
cpulimit -p pid -l 50
```

___

##### Tool: [pwdx](https://www.cyberciti.biz/faq/unix-linux-pwdx-command-examples-usage-syntax/)

###### Show current working directory of a process

```bash
pwdx <pid>
```

___

##### Tool: [taskset](https://www.cyberciti.biz/faq/taskset-cpu-affinity-command/)

###### Start a command on only one CPU core

```bash
taskset -c 0 <command>
```

___

##### Tool: [tr](https://en.wikipedia.org/wiki/Tr_(Unix))

###### Show directories in the PATH, one per line

```bash
tr : '\n' <<<$PATH
```

___

##### Tool: [chmod](https://en.wikipedia.org/wiki/Chmod)

###### Remove executable bit from all files in the current directory

```bash
chmod -R -x+X *
```

###### Restore permission for /bin/chmod

```bash
# 1:
cp /bin/ls chmod.01
cp /bin/chmod chmod.01
./chmod.01 700 file

# 2:
/bin/busybox chmod 0700 /bin/chmod

# 3:
setfacl --set u::rwx,g::---,o::--- /bin/chmod
```

___

##### Tool: [who](https://en.wikipedia.org/wiki/Who_(Unix))

###### Find last reboot time

```bash
who -b
```

###### Detect a user sudo-su'd into the current shell

```bash
[[ $(who -m | awk '{ print $1 }') == $(whoami) ]] || echo "You are su-ed to $(whoami)"
```

___

##### Tool: [last](https://www.howtoforge.com/linux-last-command/)

###### Was the last reboot a panic?

```bash
(last -x -f $(ls -1t /var/log/wtmp* | head -2 | tail -1); last -x -f /var/log/wtmp) | \
grep -A1 reboot | head -2 | grep -q shutdown && echo "Expected reboot" || echo "Panic reboot"
```

___

##### Tool: [screen](https://en.wikipedia.org/wiki/GNU_Screen)

###### Start screen in detached mode

```bash
screen -d -m <command>
```

###### Attach to an existing screen session

```bash
screen -r -d <pid>
```

___

##### Tool: [script](https://en.wikipedia.org/wiki/Script_(Unix))

###### Record and replay terminal session

```bash
### Record session
# 1)
script -t 2>~/session.time -a ~/session.log

# 2)
script --timing=session.time session.log

### Replay session
scriptreplay --timing=session.time session.log
```

___

##### Tool: [du](https://en.wikipedia.org/wiki/GNU_Screen)

###### Show 20 biggest directories with 'K M G'

```bash
du | \
sort -r -n | \
awk '{split("K M G",v); s=1; while($1>1024){$1/=1024; s++} print int($1)" "v[s]"\t"$2}' | \
head -n 20
```

___

##### Tool: [inotifywait](https://en.wikipedia.org/wiki/GNU_Screen)

###### Init tool everytime a file in a directory is modified

```bash
while true ; do inotifywait -r -e MODIFY dir/ && ls dir/ ; done;
```

___

