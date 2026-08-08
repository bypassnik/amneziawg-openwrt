# AmneziaWG для OpenWrt

Форк [YAAWG](https://github.com/this-username-has-been-taken/amneziawg-openwrt) под сборку пакетов AmneziaWG 3.x для стабильных OpenWrt **24.10.x** (`.ipk`) и **25.12.x** (`.apk`).

Публикуются только:

| Пакет | Назначение |
| --- | --- |
| `amneziawg-tools` | CLI `awg`, netifd-протокол `/lib/netifd/proto/amneziawg.sh`, watchdog |
| `amneziawg-go` | Userspace-демон (TUN), ~1.3–1.4 MiB |
| `kmod-amneziawg` | Модуль ядра (~45 KiB), требует совпадения **vermagic** |

**Не публикуются:** `luci-proto-amneziawg` и сборки OpenWrt SNAPSHOT.

Репозиторий: [bypassnik/amneziawg-openwrt](https://github.com/bypassnik/amneziawg-openwrt). Upstream для подтягивания правок: `upstream` → YAAWG.

---

## kmod или go?

| | `kmod-amneziawg` | `amneziawg-go` |
| --- | --- | --- |
| Размер | ~45 KiB | ~1.3–1.4 MiB + `kmod-tun` |
| Производительность | Выше | Ниже |
| Совместимость | Только с тем же ядром (vermagic) | Достаточно совпадения `DISTRIB_ARCH` |
| Когда ставить | Прошивка того же OpenWrt patch + target, что в имени пакета | Fallback, если kmod не встаёт |

`amneziawg.sh` сам выбирает режим: если загружен модуль `amneziawg` — kernel; иначе, при наличии `/usr/bin/amneziawg-go` — userspace.

**vermagic** — отпечаток ядра. Модуль, собранный под другое ядро, не загрузится (`Invalid module format`), даже если обе прошивки «24.10».

Пример: пакет `kmod-amneziawg_*_24.10.8_*_mediatek_filogic.ipk` обычно встаёт на **24.10.8** / `mediatek/filogic`. На **24.10.7** или другом target — часто нет → ставьте `amneziawg-go`.

Проверка на роутере:

```bash
uname -r
modinfo amneziawg 2>/dev/null | grep vermagic
# или у любого штатного kmod:
modinfo mac80211 2>/dev/null | grep vermagic
```

Всегда нужен **`amneziawg-tools`** (настройка через `awg` / UCI).

---

## Имена файлов в Releases

Отдельные пакеты (не `tar.gz` с бинарниками):

```text
amneziawg-tools_<ver>-<rel>_openwrt_<DISTRIB_ARCH>.ipk
amneziawg-go_<ver>-<rel>_openwrt_<DISTRIB_ARCH>.ipk
kmod-amneziawg_<ver>-<rel>_<openwrt>_<DISTRIB_ARCH>_<target>_<subtarget>.ipk
```

Для OpenWrt ≥ 25.12 расширение — `.apk` вместо `.ipk`.

`DISTRIB_ARCH` смотрите так:

```bash
. /etc/openwrt_release
echo "$DISTRIB_ARCH"
```

Target/subtarget — в LuCI *Status → Overview* (Target Platform) или:

```bash
ubus call system board | jsonfilter -e '@.release.target'
```

---

## Prebuilt-матрица

| Target / subtarget | Типичный `DISTRIB_ARCH` |
| --- | --- |
| `ath79/generic` | `mips_24kc` |
| `ramips/mt7621` | `mipsel_24kc` |
| `mediatek/filogic` | `aarch64_cortex-a53` |
| `x86/64` | `x86_64` |
| `bcm27xx/bcm2711` | `aarch64_cortex-a72` |
| `ipq40xx/generic` | `arm_cortex-a7_neon-vfpv4` |
| `rockchip/armv8` | `aarch64_generic` / `aarch64_cortex-a53` |

Другой target — workflow **DIY - Build AmneziaWG from SDK** (`workflow_dispatch`).

---

## Установка

Пакеты не подписаны.

### OpenWrt 24.10.x (`opkg`)

```bash
opkg update
opkg install ./amneziawg-tools_*_openwrt_*.ipk
# предпочтительно kmod (если vermagic совпал):
opkg install ./kmod-amneziawg_*_24.10.*_*.ipk
# иначе или дополнительно как fallback:
opkg install ./amneziawg-go_*_openwrt_*.ipk
```

### OpenWrt 25.12.x (`apk`)

```bash
apk add --allow-untrusted ./amneziawg-tools_*_openwrt_*.apk
apk add --allow-untrusted ./kmod-amneziawg_*_25.12.*_*.apk
# или:
apk add --allow-untrusted ./amneziawg-go_*_openwrt_*.apk
```

---

## Настройка без LuCI (UCI)

Пример интерфейса в `/etc/config/network` (параметры AmneziaWG подставьте из своего конфига):

```
config interface 'awg0'
	option proto 'amneziawg'
	option private_key '...'
	list addresses '10.8.0.2/32'
	option awg_jc '4'
	option awg_jmin '50'
	option awg_jmax '1000'
	option awg_s1 '0'
	option awg_s2 '0'
	option awg_h1 '1'
	option awg_h2 '2'
	option awg_h3 '3'
	option awg_h4 '4'

config amneziawg_awg0 'peer1'
	option public_key '...'
	option endpoint_host 'vpn.example.com'
	option endpoint_port '51820'
	list allowed_ips '0.0.0.0/0'
	option persistent_keepalive '25'
```

Затем:

```bash
ifup awg0
awg show
```

Зону firewall (`wan`/`lan`/`vpn`) и маршрутизацию настройте отдельно под свою схему.

Параметры протокола v2/v3 (S3/S4, I1–I5, HeaderProtectionKey и др.) описаны в upstream AmneziaWG / YAAWG; UCI-опции — в `amneziawg-tools/files/amneziawg.sh`.

---

## Сборка в GitHub Actions

1. Push тега `v*` → draft release с пакетами для latest **24.10.x** и **25.12.x**.
2. Или **Release** workflow вручную (`workflow_dispatch`) с выбором версий OpenWrt и publish.
3. Свой target — **DIY - Build AmneziaWG from SDK**.

Для 24.10 при сборке `amneziawg-go` golang feed подменяется на актуальный из `openwrt/packages` (нужен Go ≥ 1.25).

---

## Локальный клон

```bash
git clone https://github.com/bypassnik/amneziawg-openwrt.git
cd amneziawg-openwrt
git remote add upstream https://github.com/this-username-has-been-taken/amneziawg-openwrt.git
```

Пакеты как OpenWrt feed:

```text
src-git awgopenwrt https://github.com/bypassnik/amneziawg-openwrt.git
```

---

## Благодарности

- [amnezia-vpn](https://github.com/amnezia-vpn) — upstream AmneziaWG
- [this-username-has-been-taken/amneziawg-openwrt](https://github.com/this-username-has-been-taken/amneziawg-openwrt) (YAAWG) — исправленный netifd-протокол, go-пакет, CI на SDK
