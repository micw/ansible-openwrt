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

## UCI: Configuring wifi 2g with 2 SSIDs

```yaml

  - name: Find wifi radio devices
    set_fact:
      wifi_2g_radio: "{% for k,v in openwrt_wireless.items() %}{%if v.config.band=='2g' %}{{k}}{%endif%}{%endfor%}"
      wifi_5g_radio: "{% for k,v in openwrt_wireless.items() %}{%if v.config.band=='5g' %}{{k}}{%endif%}{%endfor%}"

  - fail:
      msg: "Radio for wifi 2g not found"
    when: wifi_2g_radio == ""

  - name: Setup wifi 2G
    uci:
      command: set
      key: "wireless.{{ wifi_2g_radio }}"
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
        device: "{{ wifi_2g_radio }}"
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
        device: "{{ wifi_2g_radio }}"
        mode: ap
        ssid: guestapn
        encryption: psk2
        ifname: wifi-guest-2g
        network: guest
        key: NotSoSecretPassword
```