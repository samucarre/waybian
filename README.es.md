<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/samucarre/waybian/raw/main/logo/waybian-logo-oscuro.png">
    <source media="(prefers-color-scheme: light)" srcset="https://github.com/samucarre/waybian/raw/main/logo/waybian-logo-claro.png">
    <img src="https://github.com/samucarre/waybian/raw/main/logo/waybian-logo-oscuro.png" alt="WayBian" width="520">
  </picture>
</p>

# WayBian

**Debian 13 arrancando de forma nativa en un MacBook Pro 16" M3 Max: escritorio Wayland (Hyprland), aceleración gráfica, Wi‑Fi, sonido, AirDrop, HDMI externo y arranque dual con macOS.**

*English: [README.md](README.md).*

WayBian (Wayland + Debian) es un Debian 13 «trixie» montado a mano para el MacBook Pro 16" de 2023 con el Apple M3 Max (placa `j516c`, SoC `t6031`). Nadie da un Linux hecho para esta máquina: el soporte del M3 en Asahi Linux (septiembre de 2026) es un kernel inicial sin driver de gráfica ni distribución, y el instalador de Debian para Apple Silicon («Bananas») solo acepta firmware de M1/M2. WayBian junta las piezas (la cadena de arranque de Asahi, el kernel y el driver de GPU de Aurora Silicon, el sistema de Debian) y arregla lo que falta entre medias.

Este repositorio es la **presentación** del proyecto: qué es, cómo funciona, qué va y qué no, y qué piezas se han cambiado. Existe para compartir el resultado con quien quiera lo mismo y para aprender de quien sepa más. Los issues y las discusiones están abiertos.

El registro del día a día (cada cambio con su porqué, el espejo de todos los ficheros de configuración, los parches y los scripts que reconstruyen el sistema desde cero) está en otro repositorio privado, `waybian-lab`. Pide lo que necesites de él.

> **Estado (27-oct-2026):** alfa, en uso diario como máquina principal del autor desde el 22-oct-2026. Kernel `7.2.11-waybian-0.2`. Hay aristas.

## La máquina

| | |
|---|---|
| Modelo | MacBook Pro 16" (2023), `Mac15,9` |
| SoC | Apple M3 Max `t6031`: CPU de 16 núcleos (12P + 4E), GPU de 40 núcleos (G15C), 64 GB |
| Identificador de placa | `j516c` (`t6031-j516c.dts` en el kernel) |
| Disco | SSD de 1 TB: macOS 27 en ~800 GiB, raíz de Linux de 122,5 GiB (ext4), los dos en el mismo disco |
| Tamaño de página | 16 KiB (lo exigen las IOMMU de Apple; algún binario da por hecho 4 KiB) |

## Qué funciona

| Componente | Estado | Notas |
|---|---|---|
| Arranque dual con macOS | ✅ | Encender → Debian; mantener el botón → macOS. macOS intacto (su propio contenedor APFS, Full Security). |
| CPU, NVMe, memoria | ✅ | Páginas de 16 KiB. |
| **GPU** (G15C, 40 núcleos) | ✅ | Driver y Mesa de Aurora Silicon. OpenGL 4.6; el compositor gasta ~2 % de CPU. |
| Pantalla interna 3456×2234 | ✅ | Escala 2 en Hyprland. **Brillo** funciona (parche local del kernel para esta placa). |
| Teclado y su retroiluminación | ✅ | El teclado del M3 no tiene teclas para la luz: la lleva un botón de la barra. |
| Trackpad | ✅ | Gestos de tres dedos para cambiar de escritorio. |
| Wi‑Fi (BCM4388) | ✅ | Firmware extraído de macOS por el instalador (`vendorfw`); los arreglos del initramfs que hicieron falta están en [docs/hardware.md](docs/hardware.md). |
| Bluetooth | ✅ | Audio y periféricos. |
| Altavoces | ✅ | Cadena DSP de Asahi (mismo hardware que el 16" M2), vigilados por `speakersafetyd`. |
| Jack de auriculares | ✅ | Salida limitada a −12 dB (el DAC llega a +12 dB si no). |
| Micrófonos | ✅ | Perfil con beamforming. |
| Cámara | ✅ | `apple-isp`. |
| Batería y carga | ✅ | |
| Puertos USB‑C (USB 3) | ✅ | Los tres, con `deferred_probe_timeout=10` (los controladores type‑C esperan a un driver USB4 que aún no existe). |
| **Salida HDMI** | ✅ | Monitor ultrapanorámico a 3440×1440 y 60 Hz, con un `aquamarine` parcheado. 100 Hz pendiente. |
| Decodificación de vídeo por hardware (AVD) | ✅ | H.264 / HEVC / VP9 / AV1 por VA‑API (`libva-v4l2-request`): Chrome, mpv, GStreamer. Unos 5 W menos en una página de vídeo. |
| AirDrop | ✅ | En los dos sentidos con un iPhone, por el AWDL del propio chip Wi‑Fi (`omdrop`). |
| Suspensión | ❌ | Aún no hay en el M3. Cerrar la tapa bloquea la sesión. |
| Thunderbolt / USB4 | ❌ | Sin driver. |
| DisplayPort por USB‑C | ❌ | Investigado; necesita trabajo en kernel y arranque. |
| Leer el disco de macOS | ❌ | APFS cifrado por hardware; los ficheros van por red. |

Detalle por componente, con los arreglos que hicieron falta: [docs/hardware.md](docs/hardware.md).

## Cómo arranca

```
Botón de encendido
 └─ Boot ROM → iBoot (Apple, firmado)                        no se toca
     └─ Contenedor APFS «Debian» (stub, Permissive Security)   lo crea el instalador de Asahi
         └─ m1n1 etapa 1 → ESP/m1n1/boot.bin (m1n1 etapa 2 + device trees + U‑Boot)
             └─ U‑Boot (UEFI) → ESP/EFI/BOOT/BOOTAA64.EFI (GRUB)
                 └─ Linux (kernel Aurora/Asahi) + initramfs → Debian 13 → greetd → Hyprland
```

El instalador de Asahi se usa en su modo **«UEFI environment only»**: encoge macOS, crea el contenedor stub con el firmware de macOS 14.8.3, instala m1n1 + U‑Boot y una ESP, y nada más. Debian se escribe encima a mano. Los paquetes `m1n1`, `u-boot-asahi` y `asahi-platform` de Debian están bloqueados: no conocen el M3 y su hook de instalación reescribiría `boot.bin` y dejaría el Mac sin arrancar. Más en [docs/boot-chain.md](docs/boot-chain.md).

## Kernel y GPU

- **Kernel «WayBian 0.2»**: la rama `integration/aurora` de Aurora Silicon (Linux 7.2.11, estado del 26-oct-2026) más unos pocos parches locales (brillo del panel en esta placa, HDMI externo por la tarjeta `apple-drm-ext` y algún arreglo), páginas de 16 KiB, compilado en local. GRUB conserva el 7.1.13 de Asahi como entrada de emergencia y los kernels anteriores de reserva; un kernel nuevo se prueba siempre una vez con `grub-reboot` antes de ser el predeterminado.
- **GPU**: el M3 estrena generación de GPU (G15) y Asahi aún no la maneja. Antes de adoptar el driver de Aurora, el autor portó a la G15C un driver experimental de otra variante del M3 hasta tener cálculo estable (33 rondas de depuración en un día); las lecciones están en [docs/kernel-gpu.md](docs/kernel-gpu.md). Hoy: «Apple M3 Max (G15C)», 40 núcleos en 4 grupos, OpenGL 4.6, Hyprland con aceleración de verdad.
- **Vídeo**: el firmware del AVD es software libre (`aurora-silicon/avd-fw`), compilado en local e instalado por versión de kernel.

## Escritorio

Hyprland 0.55 sobre Wayland, con una forma de trabajar que conserva las costumbres de macOS:

- **Waybar portada a GTK4** (nuestro fork de la PR de upstream), con botones hechos a mano: un «puesto de trabajo» en un clic, un panel de recientes con arrastrar y soltar, contadores de mensajería (Discord, Telegram, WhatsApp, correo), escritorios remotos por grupos, un panel de Wi‑Fi que reintenta, una chuleta de atajos, ventiladores/temperaturas/consumo, luz del teclado, Tailscale, AirDrop y un pequeño reproductor de música (cliente Subsonic/Navidrome que suena por mpv).
- **Atajos de Mac**: ⌘C/⌘V/⌘X en todas partes (en la terminal se traducen a Ctrl+Shift, porque allí Ctrl+C interrumpe), ⌘F, ⌘W, ⌘T, ⌘⏎, ⌘+/⌘−, Ctrl+⌘F para pantalla completa; capturas ⌘⇧3/4/5 con la miniatura flotante; gestos de tres dedos.
- **Luz nocturna** como shader de Hyprland según la puesta de sol local (el driver de pantalla aún no expone matriz de color), que se quita de las capturas después.
- **AirDrop** desde y hacia un iPhone, con una tarjeta que se puede arrastrar desde que llega el fichero.
- **Inicio de sesión**: greetd con un greeter de Hyprland mínimo (fondo + terminal translúcida), dibujado por CPU para que funcione aunque falle la GPU, con caída a la consola si lo gráfico no aparece en 40 s.
- **GTK4 en todo**: Nautilus con portal de ficheros GTK4, Chrome con su backend GTK4, fuzzel, satty y **Postcard** como cliente de correo (compilado contra GTK 4.20 + libadwaita 1.8 en un prefijo propio, corriendo en Python 3.15).

Más, con el porqué de cada decisión: [docs/desktop.md](docs/desktop.md).

<p align="center">
  <img src="images/desktop.jpg" alt="Escritorio de WayBian" width="800"><br>
  <sub>El escritorio, con el panel de información del sistema.</sub>
</p>
<p align="center">
  <img src="images/waybar-panels.jpg" alt="Waybar y sus paneles" width="800"><br>
  <sub>La barra y algunos de sus paneles.</sub>
</p>
<p align="center">
  <img src="images/airdrop.jpg" alt="AirDrop" width="400">
  <img src="images/login.jpg" alt="Pantalla de inicio de sesión" width="400"><br>
  <sub>AirDrop desde un iPhone · la pantalla de inicio de sesión.</sub>
</p>

## Las piezas que se han cambiado

Cada programa que ha necesitado un cambio tiene un fork con su rama, para poder reconstruir el sistema sin esperar a upstream:

| Programa | Fork / rama | Qué cambia |
|---|---|---|
| Waybar | [samucarre/Waybar](https://github.com/samucarre/Waybar) `gtk4-debian13-m3` | port a GTK4 (PR Alexays/Waybar#5366) + arreglos: un segfault en `hyprland/workspaces`, menús de los módulos, sobrevivir al quitar/poner un monitor, `align` en los módulos custom |
| aquamarine | [samucarre/aquamarine](https://github.com/samucarre/aquamarine) `hdmi-apple-drm-ext` | renderer GBM para una tarjeta DRM sin nodo de render (el HDMI del M3) |
| omdrop-awdl | [samucarre/omdrop-awdl](https://github.com/samucarre/omdrop-awdl) `discoverable-rapido` | visible/invisible más rápido desde la barra |
| Postcard | [samucarre/postcard](https://github.com/samucarre/postcard) `debian13-m3` | OAuth de Google con refresh token guardado (sin GNOME Online Accounts), IMAP IDLE en Python < 3.14, una rareza del `SEARCH ALL` de iCloud, interruptor «agrupar conversaciones», arreglo del sandbox de WebKit con un driver GL que no es Mesa, receta para Debian 13 |
| libva-v4l2_request | [samucarre/libva-v4l2_request](https://github.com/samucarre/libva-v4l2_request) | espejo (driver VA‑API del AVD) |
| xdg-desktop-portal-gtk4 | [samucarre/xdg-desktop-portal-gtk4](https://github.com/samucarre/xdg-desktop-portal-gtk4) | espejo (diálogos de ficheros GTK4 para Hyprland) |
| Kernel | fork privado del árbol de Aurora | parches locales, configuraciones y scripts de instalación; ver [docs/kernel-gpu.md](docs/kernel-gpu.md) |

Los forks están montados como submódulos de git en [`forks/`](forks) (cada uno fijado al commit exacto que corre en la máquina), y cada commit nuestro se puede leer también como parche en [`patches/`](patches). Lista completa, bases y recetas: [docs/forks.md](docs/forks.md).

## Cómo se hizo

El kernel y el sistema de ficheros raíz se construyeron en una máquina virtual Linux arm64, se probaron en QEMU con el mismo camino UEFI → GRUB → kernel y se escribieron en el Mac desde macOS. Desde entonces cada cambio se hace en la máquina viva y queda anotado. Reproducir el sistema es ejecutar los scripts base una vez y después un script `aplicar` idempotente; ver [docs/reproduce.md](docs/reproduce.md). Todo se ha hecho con un asistente de programación con IA de copiloto (Claude Code): el autor decide qué y por qué; el asistente lee registros, escribe los scripts y prueba; cualquier cambio de kernel, arranque o GRUB se explica antes y siempre tiene camino de vuelta.

## Lo que falta y en lo que nos vendría bien ayuda

- Suspensión, USB4/Thunderbolt y DisplayPort por USB‑C: aún no.
- HDMI a 100 Hz (el monitor y el cable lo admiten; la ruta de pantalla negocia 60 Hz).
- Cualquier cosa del `t6031`/G15 que tengamos mal.

Abre un [issue](../../issues) o una [discusión](../../discussions).

## Créditos

[Asahi Linux](https://asahilinux.org) (m1n1, U‑Boot, el instalador, el trabajo de kernel, `asahi-audio`, `speakersafetyd`), [Aurora Silicon](https://github.com/aurora-silicon) (integración del kernel del M3, driver de GPU y Mesa), [Debian](https://www.debian.org), [Fedora Asahi Remix](https://fedora-asahi-remix.org) (configuración del kernel), [Hyprland](https://hyprland.org), [Waybar](https://github.com/Alexays/Waybar), [omdrop](https://github.com/omdrop), [Postcard](https://github.com/gxanshu/postcard) y toda la gente cuya ingeniería inversa hace posible Linux en Apple Silicon.

## Licencia

Sin decidir todavía. La documentación y los scripts de este repositorio son del autor; los forks conservan la licencia de su upstream. Si quieres reutilizar algo, pregunta.
