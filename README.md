# Úvod

## Turris OS 3.0+

Nejprve je třeba mít aktualizován Turris OS alespoň na verzi 3.0, tedy verzi jádra _3.18.něco_.

## Úložiště

Je třeba mít k dispozici vhodné úložiště pro kontejnery. Nejlepší je SSD, nebo HDD v SATA portu, stále dobré je totéž připojené přes USB. Když není jiného zbytí, lze použít buď SD kartu (v nativním slotu pod pamětí, či v USB), nebo obyčejný flash disk v USB. U flash disku, nebo SD karty je nutné počítat s možným brzkým opotřebením a pomalou odezvou. Interní NAND paměť Turrisu nelze použít už jen vzhledem ke kapacitě, dále proto, že po opotřebení nejde vyměnit.

## Situace ohledně distribucí

Vzhledem k architektuře procesu v Turrisu (1.0 a 1.1) je bohužel výběr linuxových distribucí poměrně dost omezený a to na:

- OpenWRT,
- Turris OS, 
- a Debian PowerPCSPE, který je poměrně experimentální.

Jak OpenWRT, tak Turris OS v kontejneru mají jistě svá využití, nicméně nejzajímavější je poslední možnost, která je rozebrána dále.

# Instalace prerekvizit

0. Aktualizace seznamu balíků

     ```
    opkg update
    ```
    
1. Instalace všech LXC balíků (ano, trochu overkill)

     ```
    opkg install liblxc luci-app-lxc lxc lxc-attach lxc-autostart lxc-cgroup lxc-checkconfig lxc-clone lxc-common lxc-config lxc-configs lxc-console lxc-create lxc-destroy lxc-device lxc-execute lxc-freeze lxc-hooks lxc-info lxc-init lxc-ls lxc-lua lxc-monitor lxc-monitord lxc-snapshot lxc-start lxc-stop lxc-templates lxc-unfreeze lxc-unshare lxc-user-nic lxc-usernsexec lxc-wait
    ```
    
2. Instalace dalších potřebných balíků

    ```
    opkg install kmod-veth vim wget
    ```

# Instalace Debian PowerPCSPEPort

Nainstalujeme debootstrap:
```
opkg install debootstrap
```

Vhodné úložiště předpokládáme připojené v `/mnt/disk` - v dalších příkazech náležitě upravte dle Vaší situace.

Vytvoříme složku pro náš kontejner:
```
mkdir -p /mnt/disk/lxc-containers/debian1
cd /mnt/disk/lxc-containers/debian1
```

Vytvoříme rootfs pro kontejner:
```
debootstrap --no-check-gpg --arch=powerpcspe sid rootfs http://ftp.de.debian.org/debian-ports/
```

Vytvoříme konfigurační soubor pro kontejner:
```  
vim ./lxc-debian1.conf
```
a zkopírujeme obsah souboru `lxc-debian1.conf` a upravíme náležitě adresy, cesty, atp.

Nastavíme DNS server pro kontejner:
```
vim ./rootfs/etc/resolv.conf
```
a upravíme adresu `127.0.0.1` na IP Turrise (výchozí `192.168.1.1`)

Chceme, aby byl kontejner dostupný i v administračním rozhraní LuCI, odkud se dá také ovládat:
```
mkdir -p /lxc/debian1/
ln -s /mnt/disk/lxc-containers/debian1/lxc-debian1.conf /lxc/debian1/config
```

Kontejner spustíme a připojíme se k jeho konzoli:
```
lxc-start -n debian1 -f lxc-debian1.conf
lxc-attach -n debian1
```

a provedeme následující příkazy
```
gpg --keyserver pgpkeys.mit.edu --recv-key B4C86482705A2CE1
gpg -a --export B4C86482705A2CE1 | apt-key add -
apt-get update
apt-get install locales
```

... a máme hotovo. Nyní můžeme využívat všechny dostupné _Debianí_ balíky, které jsou dostupné. (bohužel veliké množství balíků není k dispozici, ale je to dostatečné například k nainstalování aplikace Home Assistant pomocí pip3)

# Automatické spuštění
TODO

# Poznámky
- Je možné provést debootstrap na PC a poté rootfs zkopírovat na Turris, což může být rychlejší. Zde se pracuje pouze na Turrisu kvůli jednoduchosti.
- Je možné na úložišti používat btrfs, což lze využít k tomu, že jednou vytvořený kontejner může sloužit jako základ pro další, které se vytvoří instantně a šetří to místo.

# Zdroje
- https://www.root.cz/clanky/spusteni-openhab-serveru-na-routeru-turris/
- https://wiki.debian.org/PowerPCSPEPort
- https://l3net.wordpress.com/2013/11/03/debian-virtualization-lxc-debootstrap-filesystem/
- https://wiki.debian.org/LXC
