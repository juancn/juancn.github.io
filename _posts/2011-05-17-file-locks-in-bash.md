---
layout: post
title:  "File locks in bash"
date:   2011-05-17 18:06:00 -0300
categories: post
---

For quite a while I've been looking for a portable utility that mimics [Procmail's "lockfile" command](http://www.linuxmanpages.com/man1/lockfile.1.php). I didn't need all the functionality, just for it to lock a single file and support a retry limit and sleep parameters.
I finally implemented one using Bash's "noclobber" option. I don't know if it will work correctly on NFS, but it should work fine on most filesystems. Hopefully it will be useful to some of you.

```sh
#!/bin/bash
set -e
declare SCRIPT_NAME="$(basename $0)"

function usage {
 echo "Usage: $SCRIPT_NAME [options] <lock file>"
 echo "Options"
 echo "       -r, --retries"
 echo "           limit the number of retries before giving up the lock."
 echo "       -s, --sleeptime, -<seconds>"
 echo "           number of seconds between each attempt. Defaults to 8 seconds."
 exit 1
}

#Check that at least one argument is provided
if [ $# -lt 1 ]; then usage; fi

declare RETRIES=-1
declare SLEEPTIME=8 #in seconds
#Parse options
for arg; do
 case "$arg" in
  -r|--retries) shift; 
   if [ $# -lt 2 ]; then usage; fi; 
   RETRIES="$1"; shift 
   echo "$RETRIES" | egrep -q '^-?[0-9]+$' || usage #check that it's a number
   ;;
  -s|--sleeptime) shift; 
   if [ $# -lt 2 ]; then usage; fi; 
   SLEEPTIME="$1"; shift 
   echo "$SLEEPTIME" | egrep -q '^[0-9]+$' || usage #check that it's a number
   ;;
  --) shift ; break ;;
  -[[:digit:]]*) 
   if [ $# -lt 1 ]; then usage; fi; 
   SLEEPTIME=${1:1}; shift
   echo "$SLEEPTIME" | egrep -q '^[0-9]+$' || usage #check that it's a number
   ;;
  --*) usage;; #fail on other options
 esac
done

#Check that only one argument is left
if [ $# -ne 1 ]; then usage; fi

declare lockfile="$1"
for (( i=0; $RETRIES < 0 || i < $RETRIES; i++ )); do
 if ( set -o noclobber; echo "$$" > "$lockfile") 2> /dev/null; 
 then
  exit 0
 fi
 #Wait a bit
 sleep $SLEEPTIME
done

#Failed
cat $lockfile
exit 1
```
