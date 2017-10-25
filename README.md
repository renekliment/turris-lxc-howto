# LXC kontejnery pro Turris 1.x

## Úložiště

Je potřeba mít vhodné uložiště pro kontejnery.

Nejlepší je SSD, nebo HDD v SATA portu, stále dobré je totéž připojené přes USB. 
Když není jiného zbytí, lze použít buď SD kartu (v nativním slotu pod RAM nebo v USB čtečce), nebo obyčejný flash disk v USB. 

U flash disku, nebo SD karty je nutné počítat s možným brzkým opotřebením a pomalou odezvou.

**Interní NAND paměť Turrisu nelze použít už jen vzhledem ke kapacitě, dále proto, že po opotřebení nejde vyměnit.**

Kontejnery nelze na instalovat na uložiště ve formátu FAT. Je tedy nutné použít např. btrfs, ext4 a jiné.

## Situace ohledně distribucí

Vzhledem k architektuře procesoru v Turrisu (1.0 a 1.1) je bohužel výběr linuxových distribucí poměrně dost omezený a to na:

- OpenWRT,
- Turris OS, 
- a Debian PowerPCSPE, který je poměrně experimentální.

Jak OpenWRT, tak Turris OS v kontejneru mají jistě svá využití, nicméně nejzajímavější je poslední možnost, která je rozebrána dále.

# Instalace LXC balíčků pomocí updateru

1. Vložíme tento kód do souboru /etc/updater/user.lua

     ```
    local script_options = {
	security = "Remote",
	ca = "file:///etc/ssl/updater.pem",
	crl = "file:///tmp/crl.pem",
	ocsp = false,
	pubkey = { "file:///etc/updater/keys/release.pub" }
     }
     Script("userlist-lxc", "https://api.turris.cz/updater-defs/" .. turris_version ..  "/turris/userlists/lxc.lua", script_options)
     ```
    
2. Spustíme updater.sh, aby nám nainstaloval balíčky LXC

     ```
     updater.sh
    ```
    
3. Volitelná instalace textového editoru **vim** (pro upravování souborů lze také použít [WinSCP](https://winscp.net).

    ```
    opkg install vim 
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
debootstrap --include debian-ports-archive-keyring --arch=powerpcspe sid rootfs https://deb.debian.org/debian-ports/
```

Nastavíme správný repositář pro balíčky:
```
echo "deb https://deb.debian.org/debian-ports sid main" | cat >> ./rootfs/etc/apt/sources.list
```

Vytvoříme konfigurační soubor pro kontejner a upravíme náležitě adresy, cesty, atp.:
```  
wget https://raw.githubusercontent.com/renekliment/turris-lxc-howto/master/config
vim ./config
```

Nastavíme DNS server pro kontejner:
```
vim ./rootfs/etc/resolv.conf
```
a upravíme adresu `127.0.0.1` na IP Turrise (výchozí `192.168.1.1`)

Chceme, aby byl kontejner dostupný i v administračním rozhraní LuCI, odkud se dá také ovládat:
```
ln -s /mnt/disk/lxc-containers/debian1 /srv/lxc/debian1
```

Pokud chceme, aby byly kontejnery dostupné i v LuCI, kde s nimi můžeme manipulovat, tak jsou dvě možnosti:

1. Vytvoříme symlink
```
ln -s /mnt/disk/lxc-containers/debian1 /srv/lxc/debian1
```
2. Upravíme cestu lxc.lxcpath v souboru /etc/lxc/lxc.conf


Kontejner spustíme a připojíme se k jeho konzoli:
```
lxc-start -n debian1
lxc-attach -n debian1
```

... a máme hotovo. Nyní můžeme využívat všechny dostupné _Debianí_ balíky, které jsou dostupné. (bohužel veliké množství balíků není k dispozici, ale je to dostatečné například k nainstalování aplikace Home Assistant pomocí pip3)

# Automatické spuštění

V souboru **/etc/config/lxc-auto** nastavíme jméno našeho kontejneru podle [oficiální dokumentace]( https://www.turris.cz/doc/cs/howto/lxc#spousteni_kontejneru_pri_startu).

## Známé chyby

Před pár dny se odstranil balík **apt-transport-https**

```
I: Found additional required dependencies: fdisk libaudit-common libaudit1 libbz2-1.0 libcap-ng0 libdb5.3 libdebconfclient0 libgcrypt20 libgpg-error0 liblz4-1 libncursesw5 libsemanage-common libsemanage1 libsystemd0 libudev1
I: Found additional base dependencies: dirmngr dmsetup gnupg-l10n gnupg-utils gpg gpg-agent gpg-wks-client gpg-wks-server gpgconf gpgsm libapparmor1 libassuan0 libbsd0 libcap2 libcryptsetup4 libdevmapper1.02.1 libdns-export190 libelf1 libfastjson4 libffi6 libgmp10 libgnutls30 libhogweed4 libidn11 libidn2-0 libip4tc0 libip6tc0 libiptc0 libisc-export189 libksba8 libldap-2.4-2 libldap-common liblocale-gettext-perl liblognorm5 libmnl0 libncurses5 libnetfilter-conntrack3 libnettle6 libnfnetlink0 libnpth0 libp11-kit0 libpsl5 libsasl2-2 libsasl2-modules-db libseccomp2 libsqlite3-0 libtasn1-6 libtext-charwidth-perl libtext-iconv-perl libtext-wrapi18n-perl libunistring2 libxtables12 openssl pinentry-curses xxd
I: Checking component main on https://deb.debian.org/debian-ports...
E: Couldn't find these debs: apt-transport-https
```

**Řešení**:
  * Počkat na opravený debootstrap
  * Spustit debootstrap s parametrem--exclude=apt-transport-https
  * V souboru: ''/usr/share/debootstrap/scripts/sid'' odstranit řádek 38 - konkrétně ''apt-transport-https''
  * Použít: http://deb.debian.org/debian-ports/

## Použití LXC kontejnerů

  * Částečná izolace od hlavního systému a lepší migrovatelnost - klidně udělám factory reset Turrisu, aktualizuji jak chci, ale ten serverový kontejner je stále ve stejném stavu a po instalaci Turris OS stačí pouze pár příkazů pro opětovné integrování kontejneru do systému; tohle vidím jako velikou výhodu a už jsem změnil služby tak, že pod Turris OS jsou jen ty síťové věci a služby jsou v tom LXC Debianu
  * Obecně tam jsou plné verze balíčků i základních knihoven a může být jednodušší používat balík v LXC Debianu, než se snažit kompilovat balík do OpenWRT
  * Webový server s PHP 7, což se může hodit, pokud chcete provozovat moderní aplikace, nebo chcete využít výkonový boost ve verzi 7
  * Snadno rozběháte Home Assistant v poslední verzi
  * Snadno rozběháte cokoliv, co se kompiluje, protože přímo na Turrisu máte GCC ... i když to chvíli potrvá

# Poznámky
- Je možné provést debootstrap na PC a poté rootfs zkopírovat na Turris, což může být rychlejší. Zde se pracuje pouze na Turrisu kvůli jednoduchosti.
- Je možné na úložišti používat btrfs, což lze využít k tomu, že jednou vytvořený kontejner může sloužit jako základ pro další, které se vytvoří instantně a šetří to místo.

# Zdroje
- https://www.root.cz/clanky/spusteni-openhab-serveru-na-routeru-turris/
- https://wiki.debian.org/PowerPCSPEPort
- https://l3net.wordpress.com/2013/11/03/debian-virtualization-lxc-debootstrap-filesystem/
- https://wiki.debian.org/LXC
- https://www.turris.cz/doc/cs/howto/lxc#spousteni_kontejneru_pri_startu
