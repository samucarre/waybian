# Instalar WayBian en un MacBook Pro 16" M3 Max

Cómo poner la [imagen de la release](https://github.com/samucarre/waybian/releases) en un Mac de verdad, a mano y desde macOS. Todavía no hay paquete para el instalador de Asahi, pero la imagen es un disco completo (GPT: una ESP y una raíz ext4), así que su contenido se puede copiar al Mac: la raíz a una partición nueva y los ficheros de arranque a la ESP que crea el instalador de Asahi.

> **Alfa.** Son los pasos con los que se instaló este equipo (desde macOS, a mano), aplicados a la imagen de la release. La imagen en sí solo se ha probado en QEMU; todavía no se ha instalado así en otro Mac. Haz una copia de seguridad (Time Machine) antes de empezar. Un `dd` al disco equivocado lo borra.

**Solo el MacBook Pro de 16" con M3 Max (placa `j516c`).** El `boot.bin` de la imagen lleva el árbol de dispositivos de esa placa y de ninguna otra. Los demás M3 necesitan el suyo.

**English:** [install.md](install.md).

## Qué hace falta

- El MacBook Pro 16" M3 Max con macOS, y su contraseña de administrador.
- Espacio libre para WayBian: 45 GB como mínimo. La raíz mide 40 GiB y en el primer arranque crece hasta ocupar la partición que le des, así que deja más si puedes.
- Los ficheros de la release y `zstd` (`brew install zstd`). Únelos y descomprímelos como dicen las notas de la release; sale `waybian-0.15-qemu.img` (40 GB, disperso: ocupa unos 12 GB).
- Una hora, más o menos.

## 1. El instalador de Asahi, «UEFI environment only»

En la Terminal de macOS:

```sh
curl https://alx.sh | sh
```

- Cuando pregunte cuánto espacio dejarle a macOS, encógelo lo suficiente para que quede libre el espacio de WayBian.
- Elige **«UEFI environment only (m1n1 + U-Boot + ESP)»** y ponle un nombre (por ejemplo `WayBian`).
- Sigue sus instrucciones hasta el final: apagar, mantener pulsado el botón de encendido, elegir la entrada nueva y terminar el segundo paso en recoveryOS (tu contraseña de macOS; baja la seguridad solo de ese contenedor, macOS sigue en Seguridad completa).

Crea un contenedor pequeño de arranque, una ESP de unos 500 MB con `m1n1/boot.bin` y `vendorfw/` (el firmware del Wi‑Fi, el Bluetooth y el trackpad de tu Mac), y deja libre el resto. La entrada nueva arranca hasta U‑Boot, todavía sin nada que cargar. Vuelve a macOS (mantén pulsado el botón de encendido y elige Macintosh HD) y sigue ahí. Más sobre lo que hace el instalador en [boot-chain.md](boot-chain.md).

## 2. Localizar los dos discos

Conecta la imagen sin montarla:

```sh
hdiutil attach -imagekey diskimage-class=CRawDiskImage -nomount waybian-0.15-qemu.img
```

Saca un disco, por ejemplo `/dev/disk8`, con `disk8s1` («EFI - WAYBIAN», la ESP de la imagen) y `disk8s2` («WayBian», la raíz).

Luego mira el disco interno:

```sh
diskutil list disk0
```

La ESP del instalador de Asahi es la partición `EFI` de ~500 MB que se llama `EFI - ` más las primeras letras del nombre que le diste (`EFI - WAYBI`), justo antes del espacio libre. Abajo aparece como `disk0sE`: **cambia la E por su número real**, y el `8` por el número que dio `hdiutil`.

## 3. Crear la partición Linux y escribir la raíz

Crea una partición Linux en todo el espacio libre que hay detrás de esa ESP:

```sh
sudo diskutil addPartition disk0sE %0FC63DAF-8483-4772-8E79-3D69D8477DE4% %noformat% 0
diskutil list disk0
```

La nueva sale como `Linux Filesystem`. Llámala `disk0sR`, comprueba que mide 40 GiB o más y escribe en ella la raíz. **Comprueba el número tres veces**: esto borra lo que haya en `disk0sR`.

```sh
sudo dd if=/dev/rdisk8s2 of=/dev/rdisk0sR bs=4m
```

## 4. Poner los ficheros de arranque en la ESP

Monta las dos ESP en carpetas fijas:

```sh
sudo mkdir -p /tmp/esp-mac /tmp/esp-img
sudo diskutil mount -mountPoint /tmp/esp-mac disk0sE
sudo diskutil mount -mountPoint /tmp/esp-img disk8s1
```

Guarda el `boot.bin` del instalador al lado y copia el de WayBian (m1n1 fase 2 con el árbol de dispositivos de Aurora para el J516C y U‑Boot; la pantalla nativa y la GPU lo necesitan) y GRUB:

```sh
sudo cp /tmp/esp-mac/m1n1/boot.bin /tmp/esp-mac/m1n1/boot.bin.asahi
sudo cp /tmp/esp-img/m1n1/boot.bin /tmp/esp-mac/m1n1/boot.bin
sudo mkdir -p /tmp/esp-mac/EFI/BOOT
sudo cp /tmp/esp-img/EFI/BOOT/BOOTAA64.EFI /tmp/esp-mac/EFI/BOOT/BOOTAA64.EFI
```

No toques `vendorfw/`: es el firmware de tu Mac, y WayBian lo carga de ahí en cada arranque. GRUB encuentra la raíz por el UUID del sistema de ficheros, así que no hay que escribir ningún número de partición en ningún sitio.

Desmonta y desconecta:

```sh
sudo diskutil unmount /tmp/esp-mac
sudo diskutil unmount /tmp/esp-img
hdiutil detach /dev/disk8
```

## 5. Primer arranque

Apaga y enciende. El instalador de Asahi dejó la entrada nueva como predeterminada; si arranca macOS, mantén pulsado el botón de encendido y elígela. La cadena es m1n1 → U‑Boot (3 s) → GRUB → WayBian.

En el primer arranque la raíz crece hasta ocupar su partición, y un asistente pregunta el idioma, el teclado, la zona horaria y el nombre del equipo, y crea tu cuenta.

**La GPU.** Su driver solo arranca con un firmware que reconoce. Si no reconoce el tuyo, el escritorio se dibuja por CPU y el registro del kernel enseña la huella del firmware (`journalctl -k | grep "unsupported GPU firmware"`, detrás de `sha256`). Para aceptarlo, añade `asahi.g15_accept_text_prefix=0x<los 16 primeros dígitos de ese sha256>` a la línea del kernel: una vez con `e` en el menú de GRUB para probarlo, y para siempre en la línea `linux` de la entrada de WayBian en `/etc/grub.d/40_custom`, seguido de `sudo update-grub`.

**No instales** los paquetes `m1n1`, `u-boot-asahi` ni `asahi-platform` de Debian. No conocen el M3 y su gancho reescribiría `boot.bin`. La imagen ya los bloquea.

## Volver atrás

- **Al `boot.bin` del instalador:** desde macOS, monta la ESP como en el paso 4 y `sudo cp /tmp/esp-mac/m1n1/boot.bin.asahi /tmp/esp-mac/m1n1/boot.bin`.
- **Quitarlo todo:** los cuatro pasos de `diskutil` de [boot-chain.md](boot-chain.md#undoing-everything-from-macos) borran la ESP, la partición Linux y el contenedor de arranque, y devuelven el espacio a macOS.
