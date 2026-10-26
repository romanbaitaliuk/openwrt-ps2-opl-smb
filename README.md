# OpenWrt PS2 SMB (Zero-to-Ready)

Полная инструкция, как поднять PS2 SMB-сервер на чистом OpenWrt и не ломать роутер при смене флешек.

## Что в итоге должно работать

- Роутер работает как обычно, даже без флешки.
- При вставке флешки с меткой `PS2SMB`:
  - автоматически появляется SMB-шара `PS2SMB`
  - OPL видит игры по сети.
- При извлечении флешки:
  - SMB-сервис останавливается
  - Wi-Fi и роутинг продолжают работать.

## Требования

- OpenWrt с доступом по SSH
- USB/SD накопитель в `exFAT`
- Метка раздела: `PS2SMB`
- Структура папок на накопителе:
  - `PS2SMB/CD`
  - `PS2SMB/DVD`

## 1) Установка пакетов

```sh
opkg update
opkg install kmod-fs-exfat exfatprogs samba4-server samba4-client dbus avahi-dbus-daemon libavahi-client libavahi-dbus-support
```

Если `exfatprogs` отсутствует:

```sh
opkg install exfat-mkfs exfat-fsck
```

## 2) Подготовка флешки

Если нужно форматирование на роутере (внимание: удаляет данные):

```sh
umount /dev/sda1 2>/dev/null || true
mkfs.exfat -n PS2SMB /dev/sda1
mkdir -p /mnt/PS2SMB/PS2SMB/CD /mnt/PS2SMB/PS2SMB/DVD
```

Рекомендуется форматировать в macOS/Windows как:

- `exFAT`
- `MBR`
- Label: `PS2SMB`

## 3) Настройка Samba4 для OPL

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

## 4) On-demand поведение SMB (без зависимости роутера от флешки)

`/etc/hotplug.d/block/10-mount` должен:

- монтировать как обычно накопители
- для носителя с label `PS2SMB`:
  - на `add` запускать `samba4`
  - на `remove` останавливать `samba4`

Минимальная логика для PS2 уже добавляется в существующий `10-mount`:

```sh
# on add for label PS2SMB
uci set samba4.ps2.path="/mnt/$mntpnt/PS2SMB"
uci commit samba4
/etc/init.d/samba4 start

# on remove for label PS2SMB
/etc/init.d/samba4 stop
```

## 5) Отключить конкурирующий SMB (`ksmbd`)

```sh
/etc/init.d/ksmbd stop 2>/dev/null || true
/etc/init.d/ksmbd disable 2>/dev/null || true
```

## 6) Проверка

### С флешкой вставленной

```sh
block info | grep -E '/dev/sd|PS2SMB'
mount | grep -E '/mnt/PS2SMB|/mnt/ps2'
/etc/init.d/samba4 running
netstat -lnt 2>/dev/null | grep -E ':445|:139'
smbclient -L //192.168.2.1 -N --option='client min protocol=NT1' --option='client max protocol=NT1'
```

Ожидается:

- есть `PS2SMB` в `smbclient -L`
- порты `445/139` слушаются

### Без флешки

```sh
/etc/init.d/samba4 running
```

Ожидается:

- `samba4` не запущен
- Wi-Fi/интернет на роутере продолжают работать

## 7) Настройки OPL

В OPL:

- `SMB Server Address`: `192.168.2.1`
- `SMB Share`: `PS2SMB`
- `SMB Port`: `445`
- `SMB User`: пусто
- `SMB Password`: пусто
- `ETH device start mode`: `Auto`

## 8) Ежедневное использование

- Вставил флешку -> подождал 3-5 секунд -> запускай OPL.
- Для копирования с ПК: `\\192.168.2.1\PS2SMB`
- Вытащил флешку -> SMB сам отключится.

## Траблшутинг

### Роутер падает только с одной флешкой

Обычно это питание/железо флешки, а не конфиг:

- попробуй другую флешку (лучше USB 2.0)
- используй внешний питаемый USB-хаб
- обязательно безопасно извлекай накопитель на ПК

### Ошибка OPL `300 Cannot connect to SMB server`

- проверь, что флешка вставлена и шара поднялась
- проверь `smbclient -L ...` из шага проверки
- убедись, что в OPL правильные `IP/share/port`

