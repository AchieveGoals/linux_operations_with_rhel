# Yum Operations How To - how to solve failed Yum command due to RHSM

Red Hat Subscription Manager (RHSM) may 'time out' and cause normal monitoring for Red Hat Enterprise Linux (RHEL) updates to fail.

Two quick aliases in my enviroment include ```currdate``` and ```yfc``` from excert from ~.bashrc (or ~/.bash_aliases) are:

```Bash
#// Local host
alias currdate="date +%Y%m%d-%H%M%S"
alias ls="ls"
# yum
alias yfc='yum makecache fast; yum check-update --security | grep -v excluded'
alias rhelsm='subscription-manager register --username="you@domain-at-rhaccess" --auto-attach --force'
```
As you can see, "yfc" runs 2 commands to refresh the local cached yum repositories and then check updates but show the security updates out of all possibly available package updates.

## Issue 01 - failed yum related repo cache update and yum check for security updates

Sometimes yfc fails with nasty message:

```
[my-server:larryt:nodb:~]$ alias yfc='yum makecache fast; yum check-update --security | grep -v excluded'
[my-server:larryt:nodb:~]$ yfc
Loaded plugins: product-id, search-disabled-repos, subscription-manager
There are no enabled repos.
 Run "yum repolist all" to see the repos you have.
 To enable Red Hat Subscription Management repositories:
     subscription-manager repos --enable <repo>
 To enable custom repositories:
     yum-config-manager --enable <repo>
Loaded plugins: product-id, search-disabled-repos, subscription-manager
There are no enabled repos.
 Run "yum repolist all" to see the repos you have.
 To enable Red Hat Subscription Management repositories:
     subscription-manager repos --enable <repo>
 To enable custom repositories:
     yum-config-manager --enable <repo>
[my-server:larryt:nodb:~]$ 
```

## Issue 01 - solution - if you know you have a valid subscription for this server / container / virtual machine

Note:  Using ```sudo``` with the ```subscription-manager register``` commands you can attach a subscription to your previously registered server. 

If you use Ansible, this is something you can try

```
[my-server:larryt:nodb:~]$ sudo subscription-manager register --username="you@domain-at-rhaccess" --auto-attach --force
[sudo] password for larryt:
Unregistering from: subscription.rhsm.redhat.com:443/subscription
The system with UUID {{mumble-mumble-mumble}} has been unregistered
All local data removed
Registering to: subscription.rhsm.redhat.com:443/subscription
Password:
The system has been registered with ID: 90f971bf-d888-478d-b250-39c5a42cc681
The registered system name is: my-server.my-domain.com
Installed Product Current Status:
Product Name: Red Hat Enterprise Linux Server
Status:       Subscribed

[my-server:larryt:nodb:~]$ currdate
20230918-120423
[my-server:larryt:nodb:~]$ yfc
Loaded plugins: product-id, search-disabled-repos, subscription-manager
rhel-7-server-rpms                                                             | 3.5 kB  00:00:00
(1/2): rhel-7-server-rpms/7Server/x86_64/updateinfo                            | 4.3 MB  00:00:00
(2/2): rhel-7-server-rpms/7Server/x86_64/primary_db                            |  96 MB  00:00:04
Metadata Cache Created
Loaded plugins: product-id, search-disabled-repos, subscription-manager
17 package(s) needed for security, out of 24 available

bind-export-libs.x86_64              32:9.11.4-26.P2.el7_9.14 rhel-7-server-rpms
 ...

subscription-manager.x86_64          1.24.52-2.el7_9          rhel-7-server-rpms
subscription-manager-rhsm.x86_64     1.24.52-2.el7_9          rhel-7-server-rpms
subscription-manager-rhsm-certificates.x86_64
                                     1.24.52-2.el7_9          rhel-7-server-rpms
[my-server:larryt:nodb:~]$ 
```
