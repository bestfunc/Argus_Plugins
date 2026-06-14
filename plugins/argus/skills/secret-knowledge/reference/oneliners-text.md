<!-- Vendored from https://github.com/trimstray/the-book-of-secret-knowledge (MIT License). Snapshot: 2026-06-14. 本文件是该知识库按分类拆分出来的一部分，供 Argus secret-knowledge skill 按需 Grep 检索。原始版权归 trimstray 及贡献者所有，保留 MIT 许可。 -->

##### Tool: [git](https://git-scm.com/)

###### Log alias for a decent view of your repo

```bash
# 1)
git log --oneline --decorate --graph --all

# 2)
git log --graph \
--pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' \
--abbrev-commit
```

___

##### Tool: [python](https://www.python.org/)

###### Static HTTP web server

```bash
# Python 3.x
python3 -m http.server 8000 --bind 127.0.0.1

# Python 2.x
python -m SimpleHTTPServer 8000
```

###### Static HTTP web server with SSL support

```bash
# Python 3.x
from http.server import HTTPServer, BaseHTTPRequestHandler
import ssl

httpd = HTTPServer(('localhost', 4443), BaseHTTPRequestHandler)

httpd.socket = ssl.wrap_socket (httpd.socket,
        keyfile="path/to/key.pem",
        certfile='path/to/cert.pem', server_side=True)

httpd.serve_forever()

# Python 2.x
import BaseHTTPServer, SimpleHTTPServer
import ssl

httpd = BaseHTTPServer.HTTPServer(('localhost', 4443),
        SimpleHTTPServer.SimpleHTTPRequestHandler)

httpd.socket = ssl.wrap_socket (httpd.socket,
        keyfile="path/tp/key.pem",
        certfile='path/to/cert.pem', server_side=True)

httpd.serve_forever()
```

###### Encode base64

```bash
python -m base64 -e <<< "sample string"
```

###### Decode base64

```bash
python -m base64 -d <<< "dGhpcyBpcyBlbmNvZGVkCg=="
```

##### Tool: [awk](http://www.grymoire.com/Unix/Awk.html)

###### Search for matching lines

```bash
# egrep foo
awk '/foo/' filename
```

###### Search non matching lines

```bash
# egrep -v foo
awk '!/foo/' filename
```

###### Print matching lines with numbers

```bash
# egrep -n foo
awk '/foo/{print FNR,$0}' filename
```

###### Print the last column

```bash
awk '{print $NF}' filename
```

###### Find all the lines longer than 80 characters

```bash
awk 'length($0)>80{print FNR,$0}' filename
```

###### Print only lines of less than 80 characters

```bash
awk 'length < 80' filename
```

###### Print double new lines a file

```bash
awk '1; { print "" }' filename
```

###### Print line numbers

```bash
awk '{ print FNR "\t" $0 }' filename
awk '{ printf("%5d : %s\n", NR, $0) }' filename   # in a fancy manner
```

###### Print line numbers for only non-blank lines

```bash
awk 'NF { $0=++a " :" $0 }; { print }' filename
```

###### Print the line and the next two (i=5) lines after the line matching regexp

```bash
awk '/foo/{i=5+1;}{if(i){i--; print;}}' filename
```

###### Print the lines starting at the line matching 'server {' until the line matching '}'

```bash
awk '/server {/,/}/' filename
```

###### Print multiple columns with separators

```bash
awk -F' ' '{print "ip:\t" $2 "\n port:\t" $3' filename
```

###### Remove empty lines

```bash
awk 'NF > 0' filename

# alternative:
awk NF filename
```

###### Delete trailing white space (spaces, tabs)

```bash
awk '{sub(/[ \t]*$/, "");print}' filename
```

###### Delete leading white space

```bash
awk '{sub(/^[ \t]+/, ""); print}' filename
```

###### Remove duplicate consecutive lines

```bash
# uniq
awk 'a !~ $0{print}; {a=$0}' filename
```

###### Remove duplicate entries in a file without sorting

```bash
awk '!x[$0]++' filename
```

###### Exclude multiple columns

```bash
awk '{$1=$3=""}1' filename
```

###### Substitute foo for bar on lines matching regexp

```bash
awk '/regexp/{gsub(/foo/, "bar")};{print}' filename
```

###### Add some characters at the beginning of matching lines

```bash
awk '/regexp/{sub(/^/, "++++"); print;next;}{print}' filename
```

###### Get the last hour of Apache logs

```bash
awk '/'$(date -d "1 hours ago" "+%d\\/%b\\/%Y:%H:%M")'/,/'$(date "+%d\\/%b\\/%Y:%H:%M")'/ { print $0 }' \
/var/log/httpd/access_log
```

___

##### Tool: [sed](http://www.grymoire.com/Unix/Sed.html)

###### Print a specific line from a file

```bash
sed -n 10p /path/to/file
```

###### Remove a specific line from a file

```bash
sed -i 10d /path/to/file
# alternative (BSD): sed -i'' 10d /path/to/file
```

###### Remove a range of lines from a file

```bash
sed -i <file> -re '<start>,<end>d'
```

###### Replace newline(s) with a space

```bash
sed ':a;N;$!ba;s/\n/ /g' /path/to/file

# cross-platform compatible syntax:
sed -e ':a' -e 'N' -e '$!ba' -e 's/\n/ /g' /path/to/file
```

- `:a` create a label `a`
- `N` append the next line to the pattern space
- `$!` if not the last line, ba branch (go to) label `a`
- `s` substitute, `/\n/` regex for new line, `/ /` by a space, `/g` global match (as many times as it can)

Alternatives:

```bash
# perl version (sed-like speed):
perl -p -e 's/\n/ /' /path/to/file

# bash version (slow):
while read line ; do printf "%s" "$line " ; done < file
```

###### Delete string +N next lines

```bash
sed '/start/,+4d' /path/to/file
```

___

##### Tool: [grep](http://www.grymoire.com/Unix/Grep.html)

###### Search for a "pattern" inside all files in the current directory

```bash
grep -rn "pattern"
grep -RnisI "pattern" *
fgrep "pattern" * -R
```

###### Show only for multiple patterns

```bash
grep 'INFO*'\''WARN' filename
grep 'INFO\|WARN' filename
grep -e INFO -e WARN filename
grep -E '(INFO|WARN)' filename
egrep "INFO|WARN" filename
```

###### Except multiple patterns

```bash
grep -vE '(error|critical|warning)' filename
```

###### Show data from file without comments

```bash
grep -v ^[[:space:]]*# filename
```

###### Show data from file without comments and new lines

```bash
egrep -v '#|^$' filename
```

###### Show strings with a dash/hyphen

```bash
grep -e -- filename
grep -- -- filename
grep "\-\-" filename
```

###### Remove blank lines from a file and save output to new file

```bash
grep . filename > newfilename
```

##### Tool: [perl](https://www.perl.org/)

###### Search and replace (in place)

```bash
perl -i -pe's/SEARCH/REPLACE/' filename
```

###### Edit of `*.conf` files changing all foo to bar (and backup original)

```bash
perl -p -i.orig -e 's/\bfoo\b/bar/g' *.conf
```

###### Prints the first 20 lines from `*.conf` files

```bash
perl -pe 'exit if $. > 20' *.conf
```

###### Search lines 10 to 20

```bash
perl -ne 'print if 10 .. 20' filename
```

###### Delete first 10 lines (and backup original)

```bash
perl -i.orig -ne 'print unless 1 .. 10' filename
```

###### Delete all but lines between foo and bar (and backup original)

```bash
perl -i.orig -ne 'print unless /^foo$/ .. /^bar$/' filename
```

###### Reduce multiple blank lines to a single line

```bash
perl -p -i -00pe0 filename
```

###### Convert tabs to spaces (1t = 2sp)

```bash
perl -p -i -e 's/\t/  /g' filename
```

###### Read input from a file and report number of lines and characters

```bash
perl -lne '$i++; $in += length($_); END { print "$i lines, $in characters"; }' filename
```

