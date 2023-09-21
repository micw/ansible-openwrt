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