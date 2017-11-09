# Security
Ansible Role for the all servers

## Requirements
Ansible version => 2.2

## Role tags:
  - common - role dummy tag
  - pam - set pam policy/restrictions
  - users - create/modify system users
  - system - common system security
  - knockd - setup knockd daemon
  - ssh - modify sshd config file
  - cron-apt - setup cron-apt (automatic security updates)

# Vars
## Default role variables
```yaml
knockd_sequence: '2490,8120,42380'
knockd_timeout: '3600'

sysctl_params:
    - { key: kernel.panic, value: 30 }
    - { key: net.ipv4.conf.all.accept_redirects, value: 0 }
    - { key: net.ipv6.conf.all.accept_redirects, value: 0 }
    - { key: net.ipv4.tcp_syncookies, value: 1 }
    - { key: net.ipv4.tcp_timestamps, value: 0 }
    - { key: net.ipv4.conf.all.rp_filter, value: 1 }
    - { key: net.ipv4.conf.default.rp_filter, value: 1 }
    - { key: net.ipv4.tcp_max_syn_backlog, value: 1280 }
    - { key: net.ipv4.tcp_synack_retries, value: 2 }
    - { key: net.ipv4.tcp_syn_retries, value: 5 }
    - { key: net.ipv4.tcp_keepalive_time, value: 60 }
    - { key: net.ipv4.tcp_keepalive_intvl, value: 10 }
    - { key: net.ipv4.tcp_keepalive_probes, value: 5 }
    - { key: net.ipv4.icmp_echo_ignore_broadcasts, value: 1 }

removable_users:
  - games
  - gnats
  - irc
  - list
  - news
  - proxy
  - uucp
removable_groups:
  - games
  - gnats
  - irc
  - list
  - news
  - proxy
  - uucp
nologin_users:
  - bin
  - daemon
  - libuuid
  - lp
  - mail
  - man
  - nobody
  - sys
```
## Example Playbook
```yaml
---
- hosts: all
  roles:
    - security
```
# Playbook run
## For all hosts
```
ansible-playbook -i production playbooks/security.yml
```
## Update users and firewall rules on single host
```
ansible-playbook -i develop playbooks/security.yml --tags users --limit backend-01
```
