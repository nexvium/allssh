# allssh

allssh is a small program to run a shell command on multiple hosts over ssh.  The commands are executed in parallel, so the runtime is largely independent of the number of hosts to be run on.

The program is implemented in a single file with (AFAIK) no dependencies other than perl. There is no need to install or configure anything to use it.


### Usage

    $ allssh [opts] <hosts_spec> [command]

##### Options

Options must be specified before the host spec, otherwise they are considered part of the command to be run.

##### Hosts Spec

The host_spec must be a single argument with no white space. The spec may contain commas and hyphens to specify multiple hosts. The following rules are used for host spec expansion:

* A comma followed by one or more letters or an IPv4 address denotes the end of one hostname and start of another.
* A comma followed by one or more digits that are not an IPv4 address denotes a host number to replace the number in the previous hostname.
* A hyphen preceded and followed by one or more digits indicates a number range to be expanded to generate multiple hostnames.
* An at sign followed by one or more letters indicates a named host group defined in the dot rc file.
  * A group name may be followed by `:UP` to expand to only the hosts that are up (responsive to pings).
  * A group name may be followed by `:<ATTR>` to expand to only the hosts with that attribute.
  * Only one attribute may be specified for a group.

For example, the spec `foo1,3,5-7i,192.168.0.1,@BAR` expands to the hosts `foo1i foo3i foo5i foo6i foo7i 192.168.0.1 bar1 bar2` (assuming the group BAR is composed of hosts bar1 and bar2).

Ranges are intended to be both compact and intuitive. The number of digits to the left of the hyphen determines the minimum number of digits in the expansion. The number of digits to the right of the hyphen determines the number of least-significant digits that are changed during expansion.

For example, the spec `foo8-11` expands to `foo8 foo9 foo10 foo11`; `foo08-11` expands to `foo08 foo09 foo10 foo11`; and `foo2008-11` expands to `foo2008 foo2009 foo2010 foo2011`.

Ranges must be in ascending order.

IP address ranges and IPv6 addresses are not currently supported.

Host must be configured to allow login using ssh keys. There will be no prompts for passwords.

##### Command

Care should be taken if the command to execute remotely has characters that are significant to the local shell (e.g. redirect, pipe, glob). They must either be escaped or the entire command must be enclosed in quotes so it is interpreted locally as a single argument.

Any `{}` string in the command will be replaced with the remote hostname.

### Examples

Shutdown the hosts `alpha`, `beta`, `gamma`, `delta`:

```
$ allssh -uroot alpha,beta,gamma,delta /sbin/shutdown -h now
Hosts: alpha beta delta gamma
User: root
Command: /sbin/shutdown -h now

alpha : -- NO OUTPUT --
beta  : -- NO OUTPUT --
delta : -- NO OUTPUT --
gamma : -- NO OUTPUT --
```

Report the number of GiB of RAM installed on 10 hosts named `node??i`:

```
$ allssh node01-10i 'awk "/MemTotal/ {print int(\$2/1024/1024+1)}" < /proc/meminfo'
Hosts: node01i node02i node03i node04i node05i node06i node07i node08i node09i node10i
Command: awk "/MemTotal/ {print int(\$2/1024/1024+1)}" < /proc/meminfo

node01i : 4
node02i : 4
node03i : 4
node04i : 8
node05i : 8
node06i : 8
node07i : 16
node08i : 16
node09i : 32
node10i : 32
```

Check if the current host and all hosts that are online in a host group named `CLUSTER` have their time synchronized:

```
$ allssh --sort=user localhost,@CLUSTER:UP timedatectl status
Hosts: localhost alpha delta
Command: timedatectl status

-=< localhost >=------------------------------------------------------------------------------------
                      Local time: Mon 2018-05-28 18:37:15 PDT
                  Universal time: Tue 2018-05-29 01:37:15 UTC
                        RTC time: Tue 2018-05-29 01:37:15
                       Time zone: America/Los_Angeles (PDT, -0700)
       System clock synchronized: no
systemd-timesyncd.service active: yes
                 RTC in local TZ: no
-=< alpha     >=------------------------------------------------------------------------------------
                      Local time: Mon 2018-05-28 18:37:15 PDT
                  Universal time: Tue 2018-05-29 01:37:15 UTC
                        RTC time: Tue 2018-05-29 01:37:15
                       Time zone: America/Los_Angeles (PDT, -0700)
       System clock synchronized: no
systemd-timesyncd.service active: yes
                 RTC in local TZ: no
-=< delta     >=------------------------------------------------------------------------------------
                      Local time: Mon 2018-05-28 18:37:15 PDT
                  Universal time: Tue 2018-05-29 01:37:15 UTC
                        RTC time: Tue 2018-05-29 01:37:15
                       Time zone: America/Los_Angeles (PDT, -0700)
       System clock synchronized: no
systemd-timesyncd.service active: yes
                 RTC in local TZ: no
```

### Options

The following options are supported.

* `-h` `--help`

  Output a summary of usage and options and exit.

* `-v` `--version`

  Output the version number and exit.

* `-d` `--dry-run`

  Output the ssh command that would be executed instead of running it.

* `-q` `--quiet`

  Run ssh with the `--quiet` option.

* `-nN` `--number=N`

  Run the command on N randomly-selected hosts. If N is zero (the default) the command is run on all hosts given. Specifying an N greater than the number of hosts results in an error.

* `--[no-]dedup`

  Remove, or not, duplicate hosts from the host list. Default is to dedup.

* `--[no-]wait`

  Wait, or not, for command to finish running on all hosts before displaying results. Default is to wait.

* `--[no-]times`

  Report, or not, how long it took to run the command on each host. Default is to not display.

* `--[no-]header`

  Output, or not, the header containing the expanded host list and command line.

* `-tN` `--timeout=N`

  Maximum number of seconds to wait for command to finish before giving up. If N is zero (the default) the program waits indefinitely.

* `-sMODE` `--sort=MODE`

  Sort, or not, results. Default is to sort by hostname with `--wait` and by completion order with `--no-wait`. MODE may be one of:

    * `host`: sort by hostname
    * `user`: output in order hosts are given in expanded host spec

* `-eMODE` `--exit-value=MODE`

  Report, or not, exit value from each host. Default is `auto`. MODE may be one of:

    * `never`: never report the exit value
    * `auto`: report it only if non-zero
    * `always`: always report it

* `-sMODE` `--separator=MODE`

  Output a separator, or not, between the output from each host. Default is `auto`. MODE may be one of:

    * `never`: never output separator
    * `auto`: output only when at least one output is more than one line
    * `always`: always output separator

* `-cMODE` `--color=MODE`

  Use, or not, ANSI color output. Default is to use color when writing a terminal. MODE may be one of:

    * `never`: never use color
    * `auto`: use color only when writing to a terminal
    * `always`: use color even if not writing to terminal

* `-iPATH` `--input=PATH`

  Use file as the stdin in for each host. It must be a seekable file.

* `-oPATH` `--output=PATH`

  Write command output to files instead of stdout. A dot and hostname will be appended to PATH to generate the pathname for each file.

* `-uUSERNAME` `--user=USERNAME`

  Specify username to use for ssh. Default is to not specify any username (i.e. use the caller's username or the one specified by the ssh config).

* `--ok`

  Only show output from hosts where command succeeded (exited with zero).

* `--add-host-keys`

  Automatically accept host key and continue connecting to unknown host. This should only be used in trusted environments, of course.

### RC file

If the file `~/.allsshrc` (or a pathname in the environment variable `$ALLSSHRC`) exists, it will be used to look for host group definitions.

A host group starts with a line containing the group name (and optional aliases) surrounded by brackets. Each group member follows in its own line, optionally followed by a colon and list of user-defined attributes. Instead of a hostname, a group name (prefixed by a `@`) may be specified to include all members of that group. The end of the group is indicated by EOF or the start of a new group.

For example, the file below defines three host groups.

```
[CLUSTER]
admin
@X86
@ARM

[X86,DESKTOP]
alpha : CLIENT CENTOS7
beta  : CLIENT CENTOS8

[ARM,MOBILE]
gamma : CLIENT ANDROID
delta : CLIENT IPHONE
```

### Disclaimer

Copyright (C) 2007 by Javier Alvarado.

This program is released under the MIT License. See LICENSE.md for details.
