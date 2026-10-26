# QUICKSTART (OpenWrt + PS2 OPL SMB)

## 0) Подключение

```sh
ssh root@192.168.2.1
```

## 1) Пакеты (один раз)

```sh
opkg update
opkg install kmod-fs-exfat exfatprogs samba4-server samba4-client dbus avahi-dbus-daemon libavahi-client libavahi-dbus-support
```

## 2) Samba4 для OPL (один раз)

```sh
uci -q delete samba4.@samba[0]
uci add samba4 samba >/dev/null
uci set samba4.@samba[0].workgroup='WORKGROUP'
uci set samba4.@samba[0].description='PS2 OPL SMB'
uci set samba4.@samba[0].interface='lan'
uci set samba4.@samba[0].charset='UTF-8'
uci set samba4.@samba[0].disable_netbios='0'
uci set samba4.@samba[0].allow_legacy_protocols='1'
uci set samba4.@samba[0].disable_ad_dc='1'
uci set samba4.@samba[0].disable_winbind='1'
uci -q delete samba4.ps2
uci set samba4.ps2='sambashare'
uci set samba4.ps2.name='PS2SMB'
uci set samba4.ps2.path='/mnt/PS2SMB/PS2SMB'
uci set samba4.ps2.read_only='no'
uci set samba4.ps2.guest_ok='yes'
uci set samba4.ps2.guest_only='yes'
uci set samba4.ps2.create_mask='0666'
uci set samba4.ps2.dir_mask='0777'
uci set samba4.ps2.force_root='1'
uci commit samba4
```

## 3) Выключить ksmbd (один раз)

```sh
/etc/init.d/ksmbd stop 2>/dev/null || true
/etc/init.d/ksmbd disable 2>/dev/null || true
```

## 4) Флешка/SD

- Формат: `exFAT` + `MBR`
- Label: `PS2SMB`
- Папки:
  - `PS2SMB/CD`
  - `PS2SMB/DVD`

## 5) Проверка (с флешкой)

```sh
block info | grep -E '/dev/sd|PS2SMB'
mount | grep -E '/mnt/PS2SMB|/mnt/ps2'
/etc/init.d/samba4 running
netstat -lnt 2>/dev/null | grep -E ':445|:139'
smbclient -L //192.168.2.1 -N --option='client min protocol=NT1' --option='client max protocol=NT1'
```

## 6) OPL

- SMB Server Address: `192.168.2.1`
- SMB Share: `PS2SMB`
- SMB Port: `445`
- SMB User: empty
- SMB Password: empty

## 7) Windows

- Открыть: `\\192.168.2.1\PS2SMB`
- Копировать ISO:
  - CD игры -> `CD`
  - DVD игры -> `DVD`
