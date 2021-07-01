# Installing the application
Unbound is available from packages, so I installing from there.

```jinja
- name: Install unbound
  ansible.builtin.package:
    name: dns/unbound
    state: present
```

# Configuration files
There are 4 different files for configuration:
* localdns.conf - In this file, list all internal hostnames. Items are list items in the vars/main.yml file
* block.conf - If there are any websites that should be blocked on top of the adblock list, they can be listed here
* adblock.conf - This file contains all blocked domains. The file is only created with Ansible but not filled. There is a cronjob that will update this file
* unbound.conf - The main configuration file for Unbound

In the configuration file, there are mostly the default values are left. Queries from private IP addresses are allowed, everything else is blocked. In addition for templating the configuration files, the sample configuration is removed.

# Start and enable the service
Once all files are in place, the service can be started and enabled.

```jinja
- name: Make sure unbound is running and enabled
  ansible.builtin.service:
    name: unbound
    state: started
    enabled: yes
```

# Add cronjob to update adblock list weekly
Update the adblock list once a week using cron. This cronjob will download a current list and fill the previously created adblock.conf file so that the DNS Server will reply with an NXDOMAIN for the domains listed. It will also reload Unbound.

```jinja
- name: Update adblock list
  ansible.builtin.cron:
    name: Update Adblock list
    weekday: '6'
    hour: '4'
    minute: '20'
    user: unbound
    job: >
      fetch https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts -o - |
      grep -v "#" | grep "0.0.0.0" |
      awk '{print "local-zone: \""$2"\" always_nxdomain"}' >
      /usr/local/etc/unbound/adblock.conf &&
      kill -HUP `cat /usr/local/etc/unbound/unbound.pid`
```
