# Usage Examples

## UCI: Removing a default WIFI configurations

```yaml
  - name: Remove the default Wifi for radio0
    uci:
      command: absent
      key: wireless.default_radio0
  - name: Remove the default Wifi for radio1
    uci:
      command: absent
      key: wireless.default_radio1
```

## UCI: Configuring radio0 with 2 SSIDs

```yaml
  - name: Setup wifi 2G
    uci:
      command: set
      key: wireless.radio0
      value:
        htmode: HT40
        channel: 11
        country: DE
        disabled: 0
  - name: Setup private network
    uci:
      command: section
      key: wireless.lan_2g
      type: wifi-iface
      value:
        device: radio0
        mode: ap
        ssid: privateapn
        encryption: psk2
        ifname: wifi-lan-2g
        network: lan
        key: .....redacted....
  - name: Setup guest network
    uci:
      command: section
      key: wireless.guest_2g
      type: wifi-iface
      value:
        device: radio0
        mode: ap
        ssid: guestapn
        encryption: psk2
        ifname: wifi-guest-2g
        network: guest
        key: NotSoSecretPassword
```

## Update OpenWRT
```yaml
# requires variable openwrt_sysupgrade_url, e.g. https://downloads.openwrt.org/releases/{{ version }}/targets/ath79/generic/openwrt-{{ version }}-ath79-generic-tplink_archer-c7-v2-squashfs-sysupgrade.bin
# Includes committing changes in /etc to git prior to and post update
# and uninstalls and installs some packages after updating

- name: check version
  block:
    - name: fail if version is not defined
      ansible.builtin.fail:
        msg: "Please define version: ansible-playbook --extra-vars \"version=22.03.3\" openwrt_upgrade.yml"
      when: version is undefined
      tags: [ "prepare", "install" ]

    - name: query installed version
      ansible.builtin.shell:
        cmd: |
          source /etc/openwrt_release
          echo $DISTRIB_RELEASE
      register: openwrt_release
      changed_when: False
      tags: [ "prepare", "install" ]

    - name: Stop if version is already installed
      ansible.builtin.meta: end_play
      when: openwrt_release.stdout == version
      tags: [ "prepare", "install" ]

- name: prepare upgrade
  block:
    - name: commit everything in /etc and pushing
      ansible.builtin.shell:
        cmd: |
          cd /etc/
          git commit -a -m "Commiting everything prior to sysupgrade"
          GIT_SSH_COMMAND='ssh -i /root/.ssh/git.id' GIT_TERMINAL_PROMPT=0 git push
        chdir: /etc/
      register: git_commit_push
      changed_when: "'Committer:' in git_commit_push.stdout or ' pack-reused ' in git_commit_push.stdout"
      tags: [ "prepare" ]

    - name: Set backup_paths
      ansible.builtin.set_fact:
        remote_backup_path: "{{ openwrt_remote_tmp }}/backup-pre-sysupgrade-from-{{ openwrt_release.stdout }}.tar.gz"
        local_backup_path: "~/Downloads/backup-{{ openwrt_user_host }}-pre-sysupgrade-from-{{ openwrt_release.stdout }}.tar.gz"
      tags: [ "prepare" ]

    - name: create backup of config
      ansible.builtin.command:
        cmd: "/sbin/sysupgrade --create-backup {{ remote_backup_path | quote }}"
        creates: "{{ remote_backup_path }}"
      tags: [ "prepare" ]

    - name: download backup of config
      ansible.builtin.command:
        cmd: "{{ openwrt_scp }} {{ openwrt_user_host | quote }}:{{ remote_backup_path | quote }} {{ local_backup_path }}"
        creates: "{{ local_backup_path }}"
      delegate_to: localhost
      tags: [ "prepare" ]

    - name: download openwrt image
      ansible.builtin.command:
        creates: "{{ openwrt_remote_tmp }}/sysupgrade.bin"
      tags: [ "prepare" ]

- name: upgrade
  block:
    - name: start sysupgrade
      nohup:
        command: /sbin/sysupgrade -q {{ openwrt_remote_tmp }}/sysupgrade.bin
      tags: [ "install" ]

    - name: wait for reboot
      wait_for_connection:
        timeout: 300
        delay: 30
      tags: [ "install" ]

- name: configure router
  when: inventory_hostname == "router"
  block:
    - name: set external nameserver in resolv.conf
      ansible.builtin.copy:
        content: "nameserver 9.9.9.9"
        dest: "/etc/resolv.conf"
      tags: [ "configure" ]

    - name: Remove odhcpd-ipv6only
      opkg:
        name: odhcpd-ipv6only
        update_cache: True
        state: absent
      tags: [ "configure" ]

    - name: Install packages
      opkg:
        name: "{{ item }}"
        state: present
      with_items:
        - luci-app-unbound
        - odhcpd
        - luci-app-adblock
        - git
        - curl
      tags: [ "configure" ]

    - name: restore resolv.conf and other stuff in /etc
      ansible.builtin.shell:
        cmd: |
          cd /etc/
          git checkout resolv.conf .gitignore
          mkdir /etc/backup/
          opkg list-installed > /etc/backup/installed_packages.txt
          find . -name '*-opkg' -exec rm {} \;
          find . -name '*dnsmasq*' -exec rm {} \;
      tags: [ "configure" ]

    - name: Reload adblock lists
      ansible.builtin.shell:
        cmd: /etc/init.d/adblock reload
      tags: [ "configure" ]

- name: commit /etc and push
  ansible.builtin.shell:
    cmd: |
      cd /etc/
      if [ -d "unbound" ]; then
          git add unbound/
          git commit -m "unbound root key"
      fi
      git add .
      git commit -m "upgraded to OpenWRT {{ version }}"
      GIT_SSH_COMMAND='ssh -i /root/.ssh/git.id' GIT_TERMINAL_PROMPT=0 git push
  tags: [ "configure" ]
```
