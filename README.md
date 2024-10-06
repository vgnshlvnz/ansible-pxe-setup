# ansible-pxe-setup
This repository contains files required to setup a bare minimum functional PXE server minus those OS files. iPXE is used here. 

`handlers/main.yml`
~~~
---

- name: Reload systemd daemon
  command: systemctl daemon-reload

- name: Restart DHCP service
  systemd:
    name: dhcpd
    state: restarted

- name: Restart TFTP
  systemd:
    name: tftp
    state: restarted

- name: Restart Apache
  systemd:
    name: httpd
    state: restarted

- name: Restart Samba
  systemd:
    name: smb
    state: restarted
~~~
`tasks/main.yml`
~~~
---
- name: Install DHCP server
  dnf:
    name: dhcp-server
    state: present

- name: Deploy DHCP configuration file
  template:
    src: dhcpd.conf.j2
    dest: /etc/dhcp/dhcpd.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart DHCP service  # Correct handler name for restarting the service

- name: Include DHCP service modification tasks
  include_tasks: service.yml

- name: Enable and start DHCP service after configuration
  systemd:
    name: dhcpd
    enabled: yes
    state: started
~~~
`templates/dhcpd.conf.j2`
~~~
# DHCP configuration file

default-lease-time 600;
max-lease-time 7200;
authoritative;
log-facility local7;

# Subnet declaration for ens192 interface
{% if ens192_ip and ens192_subnet and ens192_netmask %}
subnet {{ ens192_subnet }} netmask {{ ens192_netmask }} {
  range {{ ens192_range_start }} {{ ens192_range_end }};
  option routers {{ ens192_gateway }};
  option subnet-mask {{ ens192_netmask }};
  option domain-name-servers {{ ens192_dns }};
  next-server {{ ens192_tftp_server }};
  filename "{{ ens192_bootfile }}";
}
{% else %}
# No subnet declaration for ens192
{% endif %}
~~~
`tasks/service.yml` - PS: this file was created to make sure DHCP only listens to specific interface but you do not need this file because `dhcpd` will use interface based on the interface address against the subnet configured in `dhcpd.conf`

~~~
---
- name: Copy original dhcpd.service to /etc/systemd/system/
  copy:
    src: /usr/lib/systemd/system/dhcpd.service
    dest: /etc/systemd/system/dhcpd.service
    backup: yes  # Backup the original in case you need to revert

- name: Modify dhcpd.service to bind to ens192
  lineinfile:
    path: /etc/systemd/system/dhcpd.service
    regexp: '^ExecStart='
    line: 'ExecStart=/usr/sbin/dhcpd -f -cf /etc/dhcp/dhcpd.conf -user dhcpd -group dhcpd --no-pid ens192'
  notify:
    - Reload systemd daemon
    - Restart DHCP service
~~~

Following will be contents of `apache`:
` handlers/main.yml`
~~~
---
- name: Restart DHCP
  systemd:
    name: dhcpd
    state: restarted

- name: Restart TFTP
  systemd:
    name: tftp
    state: restarted

- name: Restart Apache
  systemd:
    name: httpd
    state: restarted

- name: Restart Samba
  systemd:
    name: smb
    state: restarted
~~~
`tasks/main.yml`
~~~
- name: Install Apache HTTP server
  dnf:
    name: httpd
    state: present

- name: Ensure Apache is enabled and running
  systemd:
    name: httpd
    enabled: yes
    state: started

- name: Ensure firewall allows HTTP traffic
  firewalld:
    service: http
    permanent: yes
    state: enabled
    immediate: yes
~~~
Following will be contents of `tftp`:
`handlers/main.yml`
~~~
---
- name: Restart DHCP
  systemd:
    name: dhcpd
    state: restarted

- name: Restart TFTP
  systemd:
    name: tftp
    state: restarted

- name: Restart Apache
  systemd:
    name: httpd
    state: restarted

- name: Restart Samba
  systemd:
    name: smb
    state: restarted
~~~
`tasks/main.yml`
~~~
---
- name: Install TFTP server
  dnf:
    name: tftp-server
    state: present

- name: Ensure TFTP service is enabled and started
  systemd:
    name: tftp
    enabled: yes
    state: started

- name: Ensure firewall allows TFTP traffic
  firewalld:
    service: tftp
    permanent: yes
    state: enabled
    immediate: yes
~~~
Following will be contents of `samba`:
`handlers/main.yml`
~~~
---
- name: Restart DHCP
  systemd:
    name: dhcpd
    state: restarted

- name: Restart TFTP
  systemd:
    name: tftp
    state: restarted

- name: Restart Apache
  systemd:
    name: httpd
    state: restarted

- name: Restart Samba
  systemd:
    name: smb
    state: restarted
~~~
`tasks/main.yml`
~~~
---
- name: Install Samba server
  dnf:
    name: samba
    state: present

- name: Ensure Samba is enabled and running
  systemd:
    name: smb
    enabled: yes
    state: started

- name: Deploy Samba configuration
  template:
    src: smb.conf.j2
    dest: /etc/samba/smb.conf
  notify: Restart Samba

- name: Ensure firewall allows Samba traffic
  firewalld:
    service: samba
    permanent: yes
    state: enabled
    immediate: yes
~~~
`templates/smb.conf.j2`
~~~
[global]
   workgroup = WORKGROUP
   server string = Samba Server
   security = user
   map to guest = Bad User
   dns proxy = no

[pxe]
   path = /var/lib/tftpboot
   browseable = yes
   writable = yes
   guest ok = yes
   read only = no
~~~

Now the playbook itself, which is `pxe-serve.yml`
~~~
---
- hosts: localhost
  become: yes
  vars_files:
    - /home/vgnshlvnz/pxe-serv-setup/host_vars/localhost.yml

  roles:
    - dhcp

  tasks:
    - name: Include DHCP role
      include_role:
        name: dhcp

    - name: Include TFTP role
      include_role:
        name: tftp

    - name: Include Apache role
      include_role:
        name: apache

    - name: Include Samba role
      include_role:
        name: samba
~~~
Finally the `host_vars/localhost.yml`:
~~~
ens192_ip: 10.0.0.1
ens192_subnet: 10.0.0.0
ens192_netmask: 255.255.255.0
ens192_range_start: 10.0.0.100
ens192_range_end: 10.0.0.200
ens192_gateway: 10.0.0.1
ens192_dns: 10.0.0.1
ens192_tftp_server: 10.0.0.1
ens192_bootfile: ipxe.efi
~~~

There are certain extra steps pertaining with iso of Oracle Linux 9.4, Windows Server 2022, the answerfile to minimize user intervention is omitted as they were described in previous article. 

You might have doubt if this will work, it worked for me, for both Oracle Linux 9.4 and Windows Server 2022 with EFI boot. Oracle Linux 9.4 boot was slower than Windows, which needs investigation. I have also omitted the ipxe files which will load the necessary files required for booting. 

Reference: chatGPT
