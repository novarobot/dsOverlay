# dsOverlay
Linux overlayfs layer manager with a Bash CLI and a Perl/GTK/WebKit graphical editor. / Linux overlayfs rétegkezelő Bash CLI-vel és Perl/GTK/WebKit grafikus felülettel.


# dsOverlay EN

A Linux overlayfs layer manager with both a command-line interface and a graphical editor.

`dsOverlay` combines multiple directory layers into one writable merged view.

The project contains:

- `dsOverlay` — Bash CLI for mounting, remounting and unmounting overlayfs projects
- `dsOverlayGUI` — Perl + GTK3 + WebKit2 graphical editor for layer configuration

## Features

- Native Linux `overlayfs` backend
- Multiple read-only lower layers
- One writable upper layer
- Ordered layer hierarchy stored in a plain-text file
- Enable or disable layers without deleting them
- Reorder layers graphically
- Mount, remount and unmount from CLI or GUI
- Automatic project directory initialization
- Relative and absolute layer paths
- Mount-state display in the GUI
- Symlink-aware lookup of the companion `dsOverlay` executable
- Safe work-directory handling:
  - the work directory is never deleted while the project is mounted
  - it is removed automatically after a successful unmount
- Closing the GUI never unmounts active projects automatically
- Warning on exit if the overlay is still mounted

## How it works

A project directory typically looks like this:

```text
project/
├── hierarchia.txt
├── lower/
├── overlay/
├── overlay.work/
└── out/
```

Directory roles:

| Path | Purpose |
|---|---|
| `hierarchia.txt` | Ordered list of lower layers |
| `lower/` | Default lower layer |
| `overlay/` | Writable upper layer |
| `overlay.work/` | Internal overlayfs work directory |
| `out/` | Merged mounted view |

The resulting filesystem view is:

```text
lower layers
+
overlay/ writable upper layer
=
out/ merged view
```

Files created or modified under `out/` are written into `overlay/`.

## Layer configuration

The `hierarchia.txt` file contains one layer path per line.

Example:

```text
./lower
../dataORIG
LAYERS/V101
LAYERS/V102
```

Relative paths are interpreted relative to the project directory.

Disabled entries begin with `#`:

```text
./lower
#LAYERS/disabled_mod
LAYERS/enabled_mod
```

The file is written in human-readable bottom-to-top order. The last active entry has the highest priority among lower layers.

Example:

```text
./lower
LAYERS/V101
LAYERS/V102
```

is passed to overlayfs in this effective order:

```text
LAYERS/V102:LAYERS/V101:lower
```

## Command-line usage

General syntax:

```bash
dsOverlay [mount|remount|umount] PROJECT_DIRECTORY
```

If no command is specified, `remount` is used.

### Mount

```bash
dsOverlay mount /path/to/project
```

Mounts the merged filesystem at:

```text
/path/to/project/out
```

### Remount

```bash
dsOverlay remount /path/to/project
```

This:

1. unmounts the active overlay if necessary
2. removes the old work directory after successful unmount
3. recreates the work directory
4. reloads `hierarchia.txt`
5. mounts the project again

### Unmount

```bash
dsOverlay umount /path/to/project
```

After a successful unmount, the program removes:

```text
overlay.work/
```

The work directory is never removed while the project is still mounted.

## Graphical interface

Start without arguments:

```bash
dsOverlayGUI
```

The program opens a folder chooser.

Start with a project directory:

```bash
dsOverlayGUI /path/to/project
```

The GUI supports:

- adding a layer
- editing a layer path
- deleting a layer
- enabling or disabling a layer
- moving a layer up or down
- saving `hierarchia.txt`
- remounting the project
- unmounting the project
- displaying the current mount state

Keyboard shortcuts in the layer list:

| Key | Action |
|---|---|
| `Up` | Select previous layer |
| `Down` | Select next layer |
| `Enter` | Edit selected layer |
| `Space` | Enable or disable selected layer |
| `Delete` | Delete selected layer |

## Exit behavior

Closing `dsOverlayGUI` does **not** automatically unmount the project.

If the project is still mounted, the GUI displays a warning and lets the user decide whether to exit.

This prevents accidental unmounting of an overlay that may still be used by another process.

## Executable discovery

`dsOverlayGUI` expects the `dsOverlay` executable to be located beside the real GUI script.

Example installation:

```text
/opt/dsOverlay/
├── dsOverlay
└── dsOverlayGUI
```

A symlink may point to the GUI:

```bash
ln -s /opt/dsOverlay/dsOverlayGUI /usr/local/bin/dsOverlayGUI
```

The GUI resolves the symlink and runs:

```text
/opt/dsOverlay/dsOverlay
```

Therefore a separate symlink for `dsOverlay` is not required.

## Installation

Copy both executables into the same directory:

```bash
sudo mkdir -p /opt/dsOverlay

sudo cp dsOverlay /opt/dsOverlay/dsOverlay
sudo cp dsOverlayGUI /opt/dsOverlay/dsOverlayGUI

sudo chmod +x \
	/opt/dsOverlay/dsOverlay \
	/opt/dsOverlay/dsOverlayGUI
```

Optional GUI launcher symlink:

```bash
sudo ln -s /opt/dsOverlay/dsOverlayGUI /usr/local/bin/dsOverlayGUI
```

Optional CLI symlink:

```bash
sudo ln -s /opt/dsOverlay/dsOverlay /usr/local/bin/dsOverlay
```

## Dependencies

### CLI

Required commands:

```text
bash
sudo
mount
umount
mountpoint
readlink
find
```

The Linux kernel must support `overlayfs`.

### GUI

Required components:

```text
Perl
GTK3 Perl bindings
WebKit2GTK Perl bindings
JSON::PP
pkexec / polkit
```

On Debian, Ubuntu or Linux Mint:

```bash
sudo apt update

sudo apt install \
	libgtk3-perl \
	libglib-object-introspection-perl \
	libgtk3-webkit2-perl \
	policykit-1 \
	x11-xserver-utils
```

A terminal emulator is also recommended for non-root GUI operation:

```bash
sudo apt install xterm
```

## First run

Create an empty project directory:

```bash
mkdir -p ~/overlay-project
```

Start the GUI:

```bash
dsOverlayGUI ~/overlay-project
```

If `hierarchia.txt` does not exist, the program can create:

```text
lower/
hierarchia.txt
```

with this initial configuration:

```text
./lower
```

The CLI and GUI can also create the remaining project directories when needed.

## Example workflow

```bash
mkdir -p ~/games/example-overlay/LAYERS/mod1
mkdir -p ~/games/example-overlay/LAYERS/mod2
```

Create `hierarchia.txt`:

```text
./lower
LAYERS/mod1
LAYERS/mod2
```

Mount:

```bash
dsOverlay mount ~/games/example-overlay
```

Use the merged view:

```bash
ls ~/games/example-overlay/out
```

Unmount:

```bash
dsOverlay umount ~/games/example-overlay
```

## Permissions

Mounting overlayfs requires root privileges.

The CLI automatically requests elevated privileges using `sudo`.

The GUI may restart itself through `pkexec`, or launch the CLI in a terminal where `sudo` can request the password.

## Important notes

- Do not manually delete `overlay.work/` while the project is mounted.
- Do not use the same upper or work directory for multiple simultaneous overlays.
- `upperdir` and `workdir` must be on the same filesystem.
- The project path and layer paths must not contain `:` because overlayfs uses it as the lower-layer separator.
- Changes made under `out/` are stored in `overlay/`.
- Unmount the project before moving or deleting its internal directories.
- The GUI does not automatically unmount projects on exit.

## Internal architecture

```text
Linux overlayfs
	└── dsOverlay Bash CLI
		└── dsOverlayGUI Perl application
			├── GTK3 native windows
			├── WebKit2 embedded interface
			├── HTML/CSS/JavaScript controls
			└── JSON-based communication
```

Responsibilities:

### `dsOverlay`

- validates project paths
- creates required directories
- parses `hierarchia.txt`
- constructs the `lowerdir=` argument
- performs mount, remount and unmount operations
- manages the work directory safely

### `dsOverlayGUI`

- edits the layer hierarchy
- displays mount state
- invokes the CLI
- handles folder selection
- warns before exiting with an active mount

## Author

Juhász Bálint / `novarobot`

# dsOverlay

Linux `overlayfs` rétegkezelő parancssori felülettel és grafikus szerkesztővel.

A `dsOverlay` több könyvtárréteget egyesít egyetlen írható, összevont nézetté.

A projekt a következőket tartalmazza:

- `dsOverlay` — Bash CLI overlayfs projektek mountolásához, remountolásához és lecsatolásához
- `dsOverlayGUI` — Perl + GTK3 + WebKit2 grafikus szerkesztő a rétegkonfiguráció kezeléséhez

## Funkciók

- Natív Linux `overlayfs` háttérrendszer
- Több csak olvasható lower réteg
- Egyetlen írható upper réteg
- Egyszerű szövegfájlban tárolt, rendezett réteghierarchia
- Rétegek engedélyezése vagy letiltása törlés nélkül
- Rétegek grafikus átrendezése
- Mount, remount és umount CLI-ből vagy GUI-ból
- Projektkönyvtár automatikus inicializálása
- Relatív és abszolút rétegútvonalak
- Mountállapot kijelzése a GUI-ban
- A kapcsolódó `dsOverlay` futtatható állomány symlink-felismerő megkeresése
- Biztonságos work könyvtár-kezelés:
  - a work könyvtár soha nem törlődik, amíg a projekt mountolva van
  - sikeres umount után automatikusan eltávolításra kerül
- A GUI bezárása soha nem csatolja le automatikusan az aktív projekteket
- Kilépéskor figyelmeztet, ha az overlay még mountolva van

## Működés

Egy projektkönyvtár tipikusan így néz ki:

```text
project/
├── hierarchia.txt
├── lower/
├── overlay/
├── overlay.work/
└── out/
```

A könyvtárak szerepe:

| Útvonal | Szerep |
|---|---|
| `hierarchia.txt` | A lower rétegek rendezett listája |
| `lower/` | Alapértelmezett lower réteg |
| `overlay/` | Írható upper réteg |
| `overlay.work/` | Az overlayfs belső munkakönyvtára |
| `out/` | Az összevont, mountolt nézet |

A létrejövő fájlrendszernézet:

```text
lower rétegek
+
overlay/ írható upper réteg
=
out/ összevont nézet
```

Az `out/` alatt létrehozott vagy módosított fájlok az `overlay/` könyvtárba kerülnek.

## Rétegkonfiguráció

A `hierarchia.txt` fájl soronként egy réteg útvonalát tartalmazza.

Példa:

```text
./lower
../dataORIG
LAYERS/V101
LAYERS/V102
```

A relatív útvonalak a projektkönyvtárhoz képest értelmeződnek.

A letiltott bejegyzések `#` karakterrel kezdődnek:

```text
./lower
#LAYERS/disabled_mod
LAYERS/enabled_mod
```

A fájl ember számára olvasható, alulról felfelé értelmezett sorrendben van megadva. Az utolsó aktív bejegyzés rendelkezik a legmagasabb prioritással a lower rétegek között.

Példa:

```text
./lower
LAYERS/V101
LAYERS/V102
```

az overlayfs számára ebben a tényleges sorrendben kerül átadásra:

```text
LAYERS/V102:LAYERS/V101:lower
```

## Parancssori használat

Általános szintaxis:

```bash
dsOverlay [mount|remount|umount] PROJEKT_KÖNYVTÁR
```

Ha nincs megadva parancs, a program `remount` műveletet használ.

### Mount

```bash
dsOverlay mount /eleresi/ut/projekt
```

Az összevont fájlrendszert ide mountolja:

```text
/eleresi/ut/projekt/out
```

### Remount

```bash
dsOverlay remount /eleresi/ut/projekt
```

Ez a következőket végzi el:

1. szükség esetén lecsatolja az aktív overlayt
2. sikeres umount után eltávolítja a régi work könyvtárat
3. újra létrehozza a work könyvtárat
4. újra beolvassa a `hierarchia.txt` fájlt
5. ismét mountolja a projektet

### Umount

```bash
dsOverlay umount /eleresi/ut/projekt
```

Sikeres umount után a program eltávolítja ezt:

```text
overlay.work/
```

A work könyvtár soha nem kerül eltávolításra, amíg a projekt még mountolva van.

## Grafikus felület

Indítás paraméterek nélkül:

```bash
dsOverlayGUI
```

A program megnyit egy könyvtárválasztó ablakot.

Indítás projektkönyvtárral:

```bash
dsOverlayGUI /eleresi/ut/projekt
```

A GUI a következőket támogatja:

- réteg hozzáadása
- rétegútvonal módosítása
- réteg törlése
- réteg engedélyezése vagy letiltása
- réteg mozgatása felfelé vagy lefelé
- a `hierarchia.txt` mentése
- a projekt remountolása
- a projekt lecsatolása
- az aktuális mountállapot megjelenítése

Billentyűparancsok a réteglistában:

| Billentyű | Művelet |
|---|---|
| `Fel` | Előző réteg kijelölése |
| `Le` | Következő réteg kijelölése |
| `Enter` | Kijelölt réteg módosítása |
| `Space` | Kijelölt réteg engedélyezése vagy letiltása |
| `Delete` | Kijelölt réteg törlése |

## Kilépési viselkedés

A `dsOverlayGUI` bezárása **nem** csatolja le automatikusan a projektet.

Ha a projekt még mountolva van, a GUI figyelmeztetést jelenít meg, és lehetővé teszi a felhasználó számára, hogy eldöntse, ennek ellenére kilép-e.

Ez megakadályozza egy olyan overlay véletlen lecsatolását, amelyet egy másik folyamat még használhat.

## A futtatható állomány megkeresése

A `dsOverlayGUI` elvárja, hogy a `dsOverlay` futtatható állomány a GUI script valódi helye mellett legyen.

Példa telepítés:

```text
/opt/dsOverlay/
├── dsOverlay
└── dsOverlayGUI
```

A GUI-ra mutathat symlink:

```bash
ln -s /opt/dsOverlay/dsOverlayGUI /usr/local/bin/dsOverlayGUI
```

A GUI feloldja a symlinket, és ezt futtatja:

```text
/opt/dsOverlay/dsOverlay
```

Ezért a `dsOverlay` számára nem szükséges külön symlink létrehozása.

## Telepítés

Másold mindkét futtatható állományt ugyanabba a könyvtárba:

```bash
sudo mkdir -p /opt/dsOverlay

sudo cp dsOverlay /opt/dsOverlay/dsOverlay
sudo cp dsOverlayGUI /opt/dsOverlay/dsOverlayGUI

sudo chmod +x \
	/opt/dsOverlay/dsOverlay \
	/opt/dsOverlay/dsOverlayGUI
```

Opcionális GUI-indító symlink:

```bash
sudo ln -s /opt/dsOverlay/dsOverlayGUI /usr/local/bin/dsOverlayGUI
```

Opcionális CLI symlink:

```bash
sudo ln -s /opt/dsOverlay/dsOverlay /usr/local/bin/dsOverlay
```

## Függőségek

### CLI

Szükséges parancsok:

```text
bash
sudo
mount
umount
mountpoint
readlink
find
```

A Linux kernelnek támogatnia kell az `overlayfs` fájlrendszert.

### GUI

Szükséges összetevők:

```text
Perl
GTK3 Perl kötések
WebKit2GTK Perl kötések
JSON::PP
pkexec / polkit
```

Debian, Ubuntu vagy Linux Mint rendszeren:

```bash
sudo apt update

sudo apt install \
	libgtk3-perl \
	libglib-object-introspection-perl \
	libgtk3-webkit2-perl \
	policykit-1 \
	x11-xserver-utils
```

Normál felhasználói joggal futó GUI esetén terminálemulátor telepítése is ajánlott:

```bash
sudo apt install xterm
```

## Első indítás

Hozz létre egy üres projektkönyvtárat:

```bash
mkdir -p ~/overlay-project
```

Indítsd el a GUI-t:

```bash
dsOverlayGUI ~/overlay-project
```

Ha a `hierarchia.txt` nem létezik, a program létrehozhatja ezeket:

```text
lower/
hierarchia.txt
```

ezzel a kezdeti konfigurációval:

```text
./lower
```

A CLI és a GUI szükség esetén a projekt többi könyvtárát is létre tudja hozni.

## Példa munkafolyamat

```bash
mkdir -p ~/games/example-overlay/LAYERS/mod1
mkdir -p ~/games/example-overlay/LAYERS/mod2
```

Hozd létre a `hierarchia.txt` fájlt:

```text
./lower
LAYERS/mod1
LAYERS/mod2
```

Mount:

```bash
dsOverlay mount ~/games/example-overlay
```

Az összevont nézet használata:

```bash
ls ~/games/example-overlay/out
```

Umount:

```bash
dsOverlay umount ~/games/example-overlay
```

## Jogosultságok

Az overlayfs mountolása root jogosultságot igényel.

A CLI automatikusan emelt jogosultságot kér a `sudo` használatával.

A GUI újraindíthatja önmagát `pkexec` segítségével, vagy terminálban indíthatja el a CLI-t, ahol a `sudo` bekérheti a jelszót.

## Fontos megjegyzések

- Ne töröld kézzel az `overlay.work/` könyvtárat, amíg a projekt mountolva van.
- Ne használd ugyanazt az upper vagy work könyvtárat több, egyidejűleg aktív overlayhez.
- Az `upperdir` és a `workdir` ugyanazon a fájlrendszeren kell legyen.
- A projekt útvonala és a rétegútvonalak nem tartalmazhatnak `:` karaktert, mert az overlayfs ezt használja a lower rétegek elválasztására.
- Az `out/` alatt végzett módosítások az `overlay/` könyvtárban tárolódnak.
- A projekt belső könyvtárainak mozgatása vagy törlése előtt csatold le a projektet.
- A GUI kilépéskor nem csatolja le automatikusan a projekteket.

## Belső felépítés

```text
Linux overlayfs
	└── dsOverlay Bash CLI
		└── dsOverlayGUI Perl alkalmazás
			├── GTK3 natív ablakok
			├── WebKit2 beágyazott felület
			├── HTML/CSS/JavaScript vezérlők
			└── JSON-alapú kommunikáció
```

Feladatok:

### `dsOverlay`

- ellenőrzi a projektútvonalakat
- létrehozza a szükséges könyvtárakat
- feldolgozza a `hierarchia.txt` fájlt
- összeállítja a `lowerdir=` argumentumot
- végrehajtja a mount, remount és umount műveleteket
- biztonságosan kezeli a work könyvtárat

### `dsOverlayGUI`

- szerkeszti a réteghierarchiát
- megjeleníti a mountállapotot
- meghívja a CLI-t
- kezeli a könyvtárválasztást
- aktív mount esetén kilépés előtt figyelmeztet

## Szerző

Juhász Bálint / `novarobot`
