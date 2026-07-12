# dsOverlay és dsOverlayGUI dokumentáció

## 1. Áttekintés

A `dsOverlay` rendszer célja, hogy egy projektmappán belül több különálló fájlrendszer-réteget egyetlen közös, eredő nézetként lehessen használni Linux `overlayfs` segítségével.

A rendszer két fő programból áll:

```text
dsOverlay       konzolos Bash alapú overlay kezelő eszköz
dsOverlayGUI    Perl + GTK + WebKit alapú grafikus szerkesztő és vezérlő
```

A `dsOverlay` végzi a tényleges Linux overlay mount / remount / umount műveleteket.

A `dsOverlayGUI` egy kényelmes grafikus szerkesztő a `hierarchia.txt` fájlhoz, és innen is meghívható a `dsOverlay` konzolos eszköz.

A rendszer alapgondolata:

```text
több alsó olvasási réteg
+
egy felső írható réteg
=
egy közös eredő out mappa
```

A projektmappa tipikus felépítése:

```text
projekt/
├── hierarchia.txt
├── lower/
├── overlay/
├── overlay.work/
└── out/
```

A mappák szerepe:

```text
hierarchia.txt     a lower rétegek sorrendjét tartalmazza
lower/             alapértelmezett első lower réteg, ha még nincs semmi
overlay/           upperdir, ide kerülnek az out-ban végzett módosítások
overlay.work/      overlayfs belső workdir mappája
out/               az eredő mountolt nézet
```

A `hierarchia.txt` soronként tartalmazza a lower rétegeket. Ezek lehetnek relatív vagy abszolút útvonalak.

Példa:

```text
./lower
../dataORIG
LAYERS/V101
/Data500/modok/valami
```

A fájlban az üres sorok kihagyásra kerülnek, a `#` karakterrel kezdődő sorok pedig tiltott / kommentelt sorokként viselkednek.

Példa:

```text
./lower
#LAYERS/kikapcsolt_mod
LAYERS/aktiv_mod
```

Ebben az esetben a `LAYERS/kikapcsolt_mod` nem kerül be az overlay lower rétegei közé.

---

## 2. Fontos overlayfs működési elv

A Linux `overlayfs` a rétegeket nem úgy kezeli, mint egy tetszőleges, középre írható hierarchiát. A kernel overlayfs működése fix:

```text
lowerdir   olvasási rétegek
upperdir   egyetlen felső írható réteg
workdir    belső munkamappa
mountpoint eredő nézet
```

A `upperdir` mindig a legfelső írható réteg, és egyben a legfelső látható réteg is. Ez azt jelenti, hogy ha az `out/` mappában módosítasz vagy létrehozol fájlokat, azok az `overlay/` mappába kerülnek.

Ez a rendszer tehát nem egy saját FUSE alapú union filesystem, hanem a Linux kernel natív overlayfs megoldását használja.

---

# 3. A projektmappa szerkezete

## 3.1 Kötelező és automatikusan létrehozott elemek

A projektmappa maga előre létező könyvtár kell legyen. A programok ezen belül dolgoznak.

A konzolos `dsOverlay` és a GUI is képes létrehozni a hiányzó alapfájlokat és mappákat.

Alapértelmezett beállítások:

```text
DEFAULT_HIERARCHY_LINE = ./lower
DEFAULT_LOWER_DIR      = ./lower
```

Ez azt jelenti, hogy ha nincs `hierarchia.txt`, akkor a program rákérdez, majd létrehozza:

```text
projekt/lower/
projekt/hierarchia.txt
```

A létrejövő `hierarchia.txt` tartalma:

```text
./lower
```

Fontos szabály:

```text
Először mindig a lower mappa jön létre.
Csak akkor jön létre a hierarchia.txt, ha a lower mappa létrehozása sikerült.
```

Ez azért fontos, mert a `hierarchia.txt` csak akkor legyen érvényes, ha az általa hivatkozott alapértelmezett mappa valóban létezik.

---

## 3.2 A konzolos tool által létrehozott jogosultságok

A konzolos `dsOverlay` minden általa létrehozott fájlt és mappát `777` jogosultsággal hoz létre.

Ez vonatkozik ezekre:

```text
lower/
hierarchia.txt
overlay/
overlay.work/
out/
```

A cél az, hogy ha a CLI rootként fut, akkor se keletkezzen olyan projektmappa, amit később a normál felhasználó nem tud szerkeszteni.

A `777` jogosultság technikailag ezt jelenti:

```text
tulajdonos: olvasás, írás, futtatás
csoport:   olvasás, írás, futtatás
mindenki:  olvasás, írás, futtatás
```

Mappáknál ez teljes hozzáférést jelent. Fájloknál, például `hierarchia.txt` esetén a futtatható bit is rákerül. Ez nem szép biztonsági szempontból, de a jelenlegi célhoz megfelelő, mert így biztosan írható marad a projekt.

---

# 4. A hierarchia.txt működése

## 4.1 Sorformátum

A `hierarchia.txt` minden érvényes sora egy lower réteg útvonala.

Példa:

```text
./lower
../eredeti_adat
LAYERS/V101
/Data500/overlay_retegek/mod1
```

A relatív utak a projektmappához képest értelmeződnek.

Tehát ha a projektmappa:

```text
/Data500/Fallout3/dsOverlay
```

és a `hierarchia.txt` egyik sora:

```text
LAYERS/V101
```

akkor ez erre mutat:

```text
/Data500/Fallout3/dsOverlay/LAYERS/V101
```

Ha a sor:

```text
../dataORIG
```

akkor ez erre mutat:

```text
/Data500/Fallout3/dataORIG
```

---

## 4.2 Komment és tiltott sorok

A `#` karakterrel kezdődő sorokat a program kihagyja.

Példa:

```text
./lower
#LAYERS/kikapcsolt
LAYERS/bekapcsolt
```

Itt csak ezek aktívak:

```text
./lower
LAYERS/bekapcsolt
```

A GUI-ban ez úgy jelenik meg, hogy az adott sor `OFF` állapotú.

Példa GUI-listában:

```text
01  [ON ] ./lower
02  [OFF] LAYERS/kikapcsolt
03  [ON ] LAYERS/bekapcsolt
```

Ha egy mappanév ténylegesen `#` karakterrel kezdődik, akkor érdemes `./` vagy abszolút útvonallal megadni.

Példa:

```text
./#teszt_reteg
/Data500/#teszt_reteg
```

---

## 4.3 Lower rétegek sorrendje

A `hierarchia.txt` emberi szempontból alulról felfelé olvasandó.

Példa:

```text
./lower
../dataORIG
LAYERS/V101
LAYERS/V102
```

Itt az elképzelt sorrend:

```text
legalul: ./lower
felette: ../dataORIG
felette: LAYERS/V101
legfelül lowerként: LAYERS/V102
```

Az overlayfs `lowerdir=` paramétere viszont fordított sorrendet vár, ezért a konzolos script a sorokat visszafelé építi be a mount parancsba.

Emberi fájl:

```text
./lower
../dataORIG
LAYERS/V101
```

Overlayfs lowerdir:

```text
LAYERS/V101:../dataORIG:./lower
```

Így a `hierarchia.txt` olvasható marad, miközben az overlayfs is a megfelelő sorrendet kapja.

---

# 5. dsOverlay konzolos program használata

## 5.1 Alapvető használat

A `dsOverlay` programot a projektmappával kell meghívni.

Alapértelmezett működés:

```bash
dsOverlay /eleresi/ut/projekt
```

Ez ugyanaz, mint:

```bash
dsOverlay remount /eleresi/ut/projekt
```

Támogatott parancsok:

```bash
dsOverlay mount /eleresi/ut/projekt
dsOverlay remount /eleresi/ut/projekt
dsOverlay umount /eleresi/ut/projekt
```

A parancsok jelentése:

```text
mount     overlay felcsatolása, ha még nincs mountolva
remount   ha mountolva van, lecsatolja, majd újra felcsatolja
umount    overlay lecsatolása
```

---

## 5.2 Első indítás üres projektmappával

Tegyük fel, hogy van egy üres projektmappa:

```text
/Data500/projekt1/
```

Ebben még nincs `hierarchia.txt`.

Indítás:

```bash
dsOverlay /Data500/projekt1
```

A program észreveszi:

```text
Hiányzik a hierarchia.txt
```

Ezután megkérdezi:

```text
Létre akarod hozni az alapértelmezett hierarchia.txt fájlt? [i/N]:
```

Ha a válasz `n`, `nem` vagy üres Enter, akkor a program kilép.

Ha a válasz `i`, `igen`, `y` vagy `yes`, akkor létrehozza:

```text
/Data500/projekt1/lower/
/Data500/projekt1/hierarchia.txt
```

A `hierarchia.txt` tartalma:

```text
./lower
```

Ezután létrehozza, ha hiányzik:

```text
/Data500/projekt1/overlay/
/Data500/projekt1/overlay.work/
/Data500/projekt1/out/
```

Majd mountolja az overlayt az `out/` mappára.

---

## 5.3 Mount

A mount parancs:

```bash
dsOverlay mount /Data500/projekt1
```

Ez ellenőrzi a projektmappát, létrehozza a hiányzó belső mappákat, majd felcsatolja az overlayt:

```text
/Data500/projekt1/out
```

Ha az `out/` már mountolva van, hibaüzenetet ad, és javasolja a `remount` használatát.

---

## 5.4 Remount

A remount parancs:

```bash
dsOverlay remount /Data500/projekt1
```

Ez a leggyakoribb művelet.

Működése:

```text
1. ellenőrzi a projektmappát
2. ha az out mountolva van, lecsatolja
3. újraolvassa a hierarchia.txt fájlt
4. újraépíti a lowerdir sorrendet
5. újra mountolja az overlayt
```

Ez akkor hasznos, ha módosult a `hierarchia.txt`, például új réteget tettünk be vagy kikapcsoltunk egy réteget.

---

## 5.5 Umount

A lecsatolás:

```bash
dsOverlay umount /Data500/projekt1
```

Ez ellenőrzi az `out/` mappát. Ha mountolva van, lecsatolja. Ha nincs mountolva, kiírja, hogy nincs mit lecsatolni.

---

## 5.6 Sudo / root jogosultság

Az overlayfs mount műveletekhez root jogosultság kell.

Ha a konzolos toolt normál felhasználóként indítod, akkor automatikusan sudo-t kér:

```text
Rendszergazdai jogosultság kell az overlay mount művelethez.
Sudo jelszó bekérése konzolban...
```

Majd újraindítja önmagát rootként:

```bash
exec sudo -- "$0" "$@"
```

Így a felhasználónak nem kell külön `sudo dsOverlay ...` formában indítania.

---

# 6. dsOverlayGUI grafikus program használata

## 6.1 Indítás

A GUI indítható paraméter nélkül:

```bash
dsOverlayGUI
```

Vagy projektmappával:

```bash
dsOverlayGUI /Data500/projekt1
```

Paraméter nélkül először GTK mappaválasztó ablak jelenik meg, ahol ki kell választani a projektmappát.

Paraméterrel a megadott projektmappát használja.

---

## 6.2 Első indítás üres projektmappával

Ha a kiválasztott projektmappában nincs `hierarchia.txt`, a GUI figyelmeztető / kérdező ablakot jelenít meg:

```text
Ebben a projektmappában nincs hierarchia.txt.
Létrehozzam?
```

Ha a felhasználó nemet választ, visszatér a projektmappa-választó ablakba.

Ha a felhasználó a projektmappa-választó ablakon nyom Mégse gombot, a program kilép.

Ha a felhasználó igent választ, akkor a GUI megpróbálja létrehozni:

```text
./lower
hierarchia.txt
```

Fontos működés:

```text
előbb ./lower
csak sikeres lower létrehozás után hierarchia.txt
```

Ha ez normál felhasználói joggal nem sikerül, a program nem áll le azonnal. Később, a pkexec-es root újraindítás után még egyszer megpróbálja.

---

## 6.3 pkexec-es root újraindítás

A GUI indítási sorrendje szándékosan ilyen:

```text
1. projektmappa kiválasztása
2. szükség esetén első próbálkozás hierarchia.txt létrehozására
3. pkexec ellenőrzés és rootként újraindítás már a kiválasztott projektmappával
4. újabb hierarchia.txt ellenőrzés
5. GUI főablak megnyitása
```

Ez azért jó, mert ha a program rootként újraindul, akkor már nem kér újra projektmappát, hanem a korábban kiválasztott útvonallal indul.

A root újraindítás parancsa lényegében:

```text
pkexec env ... perl script /kivalasztott/projekt
```

A program átadja a grafikus környezethez szükséges változókat:

```text
DISPLAY
XAUTHORITY
WAYLAND_DISPLAY
XDG_RUNTIME_DIR
DBUS_SESSION_BUS_ADDRESS
LANG
LC_ALL
LC_CTYPE
```

X11 alatt még ezt is megpróbálja:

```bash
xhost +SI:localuser:root
```

Erre azért van szükség, hogy a rootként újraindított GTK program is tudjon grafikus ablakot nyitni.

Ha a `pkexec` nem sikerül vagy a felhasználó megszakítja, a program normál joggal folytatódik.

---

## 6.4 A főablak felépítése

A főablak valójában egy GTK ablakban futó WebKit nézet. A belső kezelőfelület HTML/CSS/JavaScript.

A főablak felső részén látszik:

```text
Projektmappa
hierarchia.txt útvonala
dsOverlay bináris útvonala
Jogosultság
Overlay mount állapot
```

Példa:

```text
Projektmappa:
/Data500/projekt1

hierarchia.txt:
/Data500/projekt1/hierarchia.txt

dsOverlay bináris:
/opt/bin/dsOverlay

Jogosultság:
root joggal fut

Overlay mount állapot:
mountolva — /Data500/projekt1/out
```

Vagy:

```text
Overlay mount állapot:
nincs mountolva — /Data500/projekt1/out
```

A mount állapotot a Perl oldal ellenőrzi a `mountpoint -q` paranccsal.

---

## 6.5 Réteglista

A középső nagy lista mutatja a `hierarchia.txt` sorait.

Példa:

```text
01  [ON ] ./lower
02  [OFF] LAYERS/kikapcsolt
03  [ON ] LAYERS/aktiv
```

A `[ON]` azt jelenti, hogy a sor aktív. Mentéskor simán így kerül a fájlba:

```text
LAYERS/aktiv
```

A `[OFF]` azt jelenti, hogy a sor kikapcsolt. Mentéskor `#` kerül elé:

```text
#LAYERS/kikapcsolt
```

---

## 6.6 GUI gombok

A főablakban látható összes vezérlőgomb HTML gomb. Ezeket a WebKitben futó HTML oldal rajzolja.

Gombok:

```text
Új elem
Módosítás
Törlés
Enable / Disable
Fel
Le
Mentés
Remount
Umount
Kilépés
```

### Új elem

Az `Új elem` gomb hatására:

```text
1. JavaScript üzenetet küld Perlnek
2. Perl GTK mappaválasztó ablakot nyit
3. kiválasztott mappa visszakerül JavaScriptbe
4. JS hozzáadja a listához
5. Perl menti a hierarchia.txt fájlt
```

Ha a kiválasztott mappa a projektmappán belül van, akkor relatív útként kerül mentésre.

Példa:

```text
/Data500/projekt1/LAYERS/V101
```

helyett:

```text
./LAYERS/V101
```

vagy:

```text
LAYERS/V101
```

a konkrét logikától függően.

### Módosítás

A kijelölt sor útvonalát lehet új mappaválasztással lecserélni.

### Törlés

A kijelölt sor törlődik a listából, majd mentésre kerül.

### Enable / Disable

A kijelölt sor aktív vagy kikapcsolt állapotba kerül.

Aktívból:

```text
[ON ] LAYERS/V101
```

kikapcsolt lesz:

```text
[OFF] LAYERS/V101
```

A fájlban ez `#` előtagként jelenik meg.

### Fel / Le

A kijelölt réteg sorrendjét lehet mozgatni.

Ez fontos, mert a rétegek sorrendje befolyásolja, hogy azonos nevű fájlok esetén melyik verzió látszik az `out/` mappában.

### Mentés

A jelenlegi listaállapot kiírása a `hierarchia.txt` fájlba.

### Remount

A GUI meghívja:

```bash
/opt/bin/dsOverlay remount /projekt/mappa
```

A művelet után a GUI újra ellenőrzi, hogy az overlay mountolva van-e.

### Umount

A GUI meghívja:

```bash
/opt/bin/dsOverlay umount /projekt/mappa
```

A művelet után a GUI újra ellenőrzi a mount állapotot.

### Kilépés

Ha volt szerkesztés az utolsó remount óta, a program rákérdez:

```text
Volt szerkesztés az utolsó remount óta.
Biztosan kilépsz remount nélkül?
```

---

## 6.7 Billentyűkezelés

A HTML/JS lista támogat billentyűket is:

```text
Fel nyíl       kijelölés felfelé
Le nyíl        kijelölés lefelé
Enter          módosítás
Space          Enable / Disable
Delete         törlés
```

Ez a JavaScript oldalon történik.

---

# 7. GUI technikai felépítése

## 7.1 Rétegek a GUI programon belül

A `dsOverlayGUI` több technológiát kapcsol össze:

```text
Perl
GTK3
WebKit2GTK
HTML
CSS
JavaScript
JSON
document.title alapú üzenetküldés
külső dsOverlay Bash script
```

A felosztás:

```text
Perl:
    programlogika
    fájlkezelés
    jogosultságkezelés
    pkexec
    GTK ablakok
    dsOverlay futtatás
    mount állapot ellenőrzés

GTK:
    főablak
    WebKit konténer
    fájlválasztó ablakok
    hiba / info / kérdező ablakok

WebKit:
    HTML felület futtatása

HTML:
    fő kezelőfelület szerkezete

CSS:
    kinézet, színek, listaformázás

JavaScript:
    lista kezelése
    gombok kezelése
    kijelölés
    billentyűk
    üzenetküldés Perl felé
```

---

## 7.2 GTK oldali elemek

A Perl program ezeket a GTK/WebKit elemeket használja:

```perl
Gtk3::Window
Gtk3::WebKit2::WebView
Gtk3::FileChooserDialog
Gtk3::MessageDialog
```

Ezek szerepe:

```text
Gtk3::Window
    a natív főablak

Gtk3::WebKit2::WebView
    ebben fut a HTML/JS kezelőfelület

Gtk3::FileChooserDialog
    projektmappa és rétegmappa kiválasztása

Gtk3::MessageDialog
    hiba, információ, igen/nem kérdések
```

A főablak létrehozását a `create_gui()` függvény végzi.

Lényegi működése:

```text
1. létrehoz egy Gtk3::Window objektumot
2. beállítja a címet és ablakméretet
3. létrehoz egy WebKit2 WebView-t
4. beköti a title változás figyelését
5. betölti a HTML oldalt
6. megjeleníti az ablakot
```

---

## 7.3 HTML/JS oldal

A HTML oldalt a Perl program belső stringként tartalmazza az `html_page()` függvényben.

A HTML fő részei:

```text
top blokk
    projektmappa
    hierarchia.txt
    dsOverlay bináris
    jogosultság
    mount állapot

main blokk
    réteglista
    gombok
    státuszmező
    rövid magyarázat
```

A lista nem natív HTML `<select>`, hanem saját `div` alapú lista.

Ennek oka, hogy a natív `<select>` GTK/WebKit téma alatt hibásan vagy rosszul láthatóan jelenítette meg a kijelölt sort. A saját `div` lista teljesen kontrollálható CSS-sel.

A lista konténere:

```html
<div id="list" tabindex="0"></div>
```

A JavaScript minden sorhoz ilyen elemet hoz létre:

```javascript
const row = document.createElement("div");
row.className = "listRow";
```

Majd osztályokat ad hozzá:

```text
listRow
disabledItem
selectedItem
```

---

## 7.4 CSS felépítés

A CSS határozza meg a sötét témát, a lista kinézetét, a státuszszíneket és a mount állapot színezését.

Főbb CSS részek:

```text
body
    sötét háttér, világos szöveg

#top
    felső információs blokk

.path
    útvonalak monospace kiírása

#main
    fő tartalmi rész

#list
    saját lista konténer

.listRow
    egy listaelem

.disabledItem
    kikapcsolt listaelem

.selectedItem
    kijelölt listaelem

.warn
    figyelmeztetés színe

.ok
    rendben állapot színe

.mountOn
    mountolva állapot

.mountOff
    nincs mountolva állapot
```

Példa logika:

```css
.mountOn
{
    color: #99ff99;
    font-weight: bold;
}

.mountOff
{
    color: #ffcc66;
    font-weight: bold;
}
```

---

## 7.5 JavaScript állapotmodell

A JavaScript oldalon van egy központi `state` objektum:

```javascript
let state =
{
    project_dir: "",
    hierarchy_file: "",
    hierarchy_dir: "",
    dsoverlay_bin: "",
    items: [],
    dirty: 0,
    dirty_since_remount: 0,
    is_root: 0,
    overlay_mounted: 0,
    overlay_out_dir: ""
};
```

Ennek mezői:

```text
project_dir
    projektmappa útvonala

hierarchy_file
    hierarchia.txt teljes útvonala

hierarchy_dir
    a hierarchia.txt mappája

dsoverlay_bin
    a dsOverlay bináris útvonala

items
    a réteglista elemei

dirty
    van-e mentetlen módosítás

dirty_since_remount
    történt-e szerkesztés az utolsó remount óta

is_root
    root joggal fut-e a GUI

overlay_mounted
    mountolva van-e az out

overlay_out_dir
    az out mappa útvonala
```

A Perl oldal minden `refresh_web()` híváskor új állapotot küld a JavaScriptnek.

---

## 7.6 Listaelemek JavaScript oldalon

Egy listaelem szerkezete:

```javascript
{
    enabled: 1,
    path: "./lower"
}
```

Vagy kikapcsolt állapotban:

```javascript
{
    enabled: 0,
    path: "LAYERS/kikapcsolt"
}
```

A `render()` függvény rajzolja újra a listát.

Minden elemnél eldönti:

```text
enabled = 1 → [ON ]
enabled = 0 → [OFF]
```

Példa megjelenítés:

```text
01  [ON ] ./lower
02  [OFF] LAYERS/test
```

---

## 7.7 JavaScript gombfüggvények

A GUI fő HTML gombjaihoz JavaScript függvények tartoznak.

Fontosabb függvények:

```text
addItem()
editItem()
deleteItem()
toggleItem()
moveUp()
moveDown()
saveItems()
remount()
umountOverlay()
quitApp()
```

Ezek közül több csak lokálisan módosítja a `state.items` tömböt, majd üzenetet küld Perlnek.

Példa:

```javascript
function saveItems()
{
    send({cmd: "save_items", items: state.items});
}
```

A `remount()`:

```javascript
function remount()
{
    send({cmd: "remount"});
}
```

A JavaScript tehát nem közvetlenül mountol. Csak parancsüzenetet küld.

---

# 8. HTML/JS és Perl közti kommunikáció

## 8.1 Miért kell külön kommunikáció?

A HTML/JS felület a WebKitben fut. A fájlkezelés, mountolás, mappaválasztás viszont Perl/GTK oldalon történik.

Ezért kell egy híd:

```text
HTML/JS gombnyomás
↓
üzenet Perlnek
↓
Perl művelet
↓
válasz / állapot vissza JavaScriptnek
```

---

## 8.2 document.title alapú üzenetküldés

A program nem WebKit script message handlert használ, mert az adott Perl/WebKit binding alatt a JSCValue kezelés problémás volt.

Ehelyett egyszerű, stabil trükköt használ:

```javascript
document.title = "DSMSG:" + msgSeq + ":" + encodeURIComponent(JSON.stringify(obj));
```

A JavaScript `send()` függvénye:

```javascript
function send(obj)
{
    msgSeq++;
    document.title = "DSMSG:" + msgSeq + ":" + encodeURIComponent(JSON.stringify(obj));
}
```

A `msgSeq` azért kell, hogy egymás után azonos tartalmú üzeneteknél is megváltozzon a title, mert a Perl oldal a title változását figyeli.

---

## 8.3 Perl oldali title figyelés

A Perl oldalon a WebKit title változására van kötve egy callback:

```perl
$webview->signal_connect('notify::title' => sub
{
    handle_title_message();

    return TRUE;
});
```

A `handle_title_message()`:

```text
1. lekéri a WebView title értékét
2. ellenőrzi, hogy DSMSG:-vel kezdődik-e
3. kiszedi belőle az URL-enkódolt JSON-t
4. dekódolja
5. átadja a handle_message() függvénynek
```

A `handle_message()` már a konkrét `cmd` mezőt vizsgálja.

Példa JSON:

```json
{
    "cmd": "remount"
}
```

Vagy:

```json
{
    "cmd": "save_items",
    "items": [
        {"enabled": 1, "path": "./lower"},
        {"enabled": 0, "path": "LAYERS/test"}
    ]
}
```

---

## 8.4 Perl → JavaScript visszairány

A Perl oldal a `run_javascript()` hívással tud JavaScriptet futtatni a WebKitben.

Segédfüggvény:

```perl
sub js_call
{
    my ($code) = @_;

    $webview->run_javascript($code, undef, undef, undef);
}
```

A `refresh_web()` létrehoz egy JSON állapotcsomagot, majd meghívja:

```javascript
window.dsSetState(...)
```

Perl oldali logika:

```perl
my $encoded = $json->encode($payload);
js_call('window.dsSetState(' . $encoded . ');');
```

JavaScript oldalon:

```javascript
window.dsSetState = function(newState)
{
    state = newState;
    render();
};
```

Így frissül a teljes GUI.

---

# 9. GTK üzenetablakok felépítése

## 9.1 Üzenetablak típusok

A GUI-ban több külön függvény van különböző GTK üzenetablak típusokra.

Főablakhoz kötött változatok:

```text
show_error()
info_gui()
ask_yes_no()
```

Főablak nélküli változatok:

```text
show_error_no_parent()
show_warning_no_parent()
ask_yes_no_no_parent()
```

A főablak nélküli változatok induláskor kellenek, amikor még nincs létrehozva a fő `$window`.

Példa:

```text
program indul
↓
projektmappa kiválasztás
↓
nincs hierarchia.txt
↓
ask_yes_no_no_parent()
```

Ilyenkor még nincs főablak, ezért a kérdezőablaknak nincs parent window-ja.

---

## 9.2 Ezek nem konkrét üzenetek, hanem ablaktípusok

Például az `info_gui()` nem azt jelenti, hogy mindig ugyanazt a szöveget írja ki. Ez csak egy információs ablak típust valósít meg.

Példa:

```perl
info_gui('Remount kész.');
```

vagy:

```perl
info_gui('Umount kész.');
```

Mindkettő ugyanazt az információs GTK ablakot használja, csak más szöveggel.

Ugyanez igaz a hibára:

```perl
show_error("Nem sikerült megnyitni a fájlt.");
show_error("A dsOverlay hibával tért vissza.");
```

---

## 9.3 Ablak eltüntetési probléma és flush_gtk_events()

Indításkor előfordult, hogy a fájlválasztó vagy kérdezőablak a háttérben „beragadt”, látszólag befagyva ott maradt.

Ennek oka, hogy a GTK `destroy()` után még nem feltétlenül dolgozta fel az ablak tényleges eltüntetését, de a program már ment tovább például `pkexec` vagy más blokkoló művelet irányába.

Erre készült a `flush_gtk_events()` függvény:

```perl
sub flush_gtk_events
{
    while (Gtk3::events_pending())
    {
        Gtk3::main_iteration();
    }
}
```

A dialogok bezárásakor ezért a helyes sorrend:

```text
hide()
destroy()
flush_gtk_events()
rövid várakozás, ha szükséges
flush_gtk_events()
```

Ez biztosítja, hogy a fájlválasztó vagy kérdezőablak tényleg eltűnjön, mielőtt a program továbbmegy.

---

# 10. Perl oldali fő függvények

## 10.1 Beállítási változók

A GUI kód elején vannak a fontos beállítások:

```perl
my $DEFAULT_HIERARCHY_LINE = './lower';
my $DEFAULT_LOWER_DIR = './lower';
my $PREFERRED_TERMINAL = '';
my $SYSTEM_TERMINAL_CANDIDATE = 'x-terminal-emulator';
my $DSOVERLAY_BIN = '/opt/bin/dsOverlay';
```

Ezek jelentése:

```text
DEFAULT_HIERARCHY_LINE
    hiányzó hierarchia.txt esetén ez kerül a fájlba

DEFAULT_LOWER_DIR
    ez a mappa jön létre először

PREFERRED_TERMINAL
    ha meg van adva, ezt próbálja először külön terminálnak

SYSTEM_TERMINAL_CANDIDATE
    Debian/Ubuntu/Mint alapértelmezett terminál

DSOVERLAY_BIN
    a konzolos dsOverlay script útvonala
```

---

## 10.2 init_paths()

Az `init_paths()` felelős az indulási útvonalak beállításáért.

Feladatai:

```text
1. ha van parancssori paraméter, azt projektmappának veszi
2. ha nincs paraméter, GTK mappaválasztót nyit
3. hiányzó hierarchia.txt esetén létrehozási logika
4. pkexec újraindítás rootként, ha lehet
5. végső hierarchia.txt ellenőrzés
6. hierarchy_file és hierarchy_dir beállítása
```

Ez a GUI indulásának egyik legfontosabb része.

---

## 10.3 try_create_default_hierarchy()

Ez hozza létre az alapértelmezett struktúrát.

Sorrend:

```text
1. ellenőrzi, van-e már hierarchia.txt
2. kiszámolja az alap lower útvonalat
3. ha nincs lower, létrehozza
4. ha a lower létrehozás nem sikerül, nem próbál txt-t létrehozni
5. ha lower már létezik, létrehozza a hierarchia.txt fájlt
6. beleírja: ./lower
```

A legfontosabb szabály:

```text
TXT csak akkor készül, ha a lower mappa már biztosan létezik.
```

---

## 10.4 restart_self_with_pkexec_if_needed()

Ez próbálja a GUI-t rootként újraindítani.

Lépések:

```text
1. ha már root, nincs teendő
2. ha már volt pkexec próbálkozás, nem próbálja újra
3. megkeresi a saját script útvonalát
4. ellenőrzi, van-e pkexec
5. X11 alatt xhost engedélyezést próbál
6. összerakja az env változókat
7. pkexec-kel újraindítja önmagát
8. argumentumként átadja a projektmappát
9. ha sikerült, az eredeti folyamat kilép
10. ha nem sikerült, normál joggal folytatja
```

Ez biztosítja, hogy a GUI lehetőség szerint rootként fusson, de ne legyen használhatatlan akkor sem, ha a felhasználó megszakítja a jogosultságkérést.

---

## 10.5 load_hierarchy()

Beolvassa a `hierarchia.txt` tartalmát a Perl `@items` tömbbe.

A sorokból ilyen elemek lesznek:

```perl
{
    enabled => 1,
    path => './lower',
}
```

Vagy kikapcsolt sor esetén:

```perl
{
    enabled => 0,
    path => 'LAYERS/test',
}
```

Az üres sorokat kihagyja. A `#`-el kezdődő sorokat disabled elemként tölti be.

---

## 10.6 save_hierarchy()

A `@items` tömb aktuális állapotát visszaírja a `hierarchia.txt` fájlba.

Ha egy elem aktív:

```text
LAYERS/V101
```

Ha kikapcsolt:

```text
#LAYERS/V101
```

Mentés után:

```text
dirty = 0
dirty_since_remount = 1
```

Ez azt jelenti, hogy nincs mentetlen módosítás, de volt szerkesztés az utolsó remount óta.

---

## 10.7 refresh_web()

Ez küldi át az aktuális állapotot a JavaScript oldalnak.

Tartalmazza:

```text
projektmappa
hierarchia.txt útvonal
réteglista
dirty állapot
root állapot
mount állapot
out mappa útvonal
```

A mount állapotot előtte lekérdezi:

```text
get_overlay_mount_status()
```

Majd meghívja:

```javascript
window.dsSetState(...)
```

---

## 10.8 get_overlay_mount_status()

Ez ellenőrzi, hogy az overlay mountolva van-e.

Logika:

```text
1. out_dir = projektmappa/out
2. ha nincs ilyen mappa, akkor nincs mountolva
3. mountpoint -q out_dir
4. ha exit code 0, mountolva
5. különben nincs mountolva
```

Ez a GUI megnyitásakor és minden remount / umount után lefut.

---

## 10.9 run_dsoverlay()

Ez hívja meg a konzolos `dsOverlay` scriptet.

Parancs:

```text
/opt/bin/dsOverlay remount /projekt
```

vagy:

```text
/opt/bin/dsOverlay umount /projekt
```

A működése eltér attól függően, hogy milyen környezetben fut:

### Rootként futó GUI

Ha a GUI rootként fut, akkor közvetlenül indítja a `dsOverlay` parancsot:

```perl
system(@cmd);
```

Nincs külön terminál, nincs sudo kérdés.

### Terminálból indított normál GUI

Ha a GUI nem root, de van szülő terminál, akkor a meglévő terminálban futtatja a parancsot. Ha sudo jelszó kell, ott lehet megadni.

### Menüikonból indított normál GUI

Ha nincs szülő terminál, akkor külön terminált próbál indítani.

Termináljelöltek:

```text
PREFERRED_TERMINAL
x-terminal-emulator
gnome-terminal
xfce4-terminal
mate-terminal
konsole
xterm
```

A terminálban lefut a dsOverlay, majd kiírja az exit code-ot, és Enterre vár, hogy ne záródjon be azonnal.

---

# 11. Konzolos dsOverlay technikai felépítése

## 11.1 Beállítási változók

A Bash script elején vannak:

```bash
DEFAULT_HIERARCHY_LINE="./lower"
DEFAULT_LOWER_DIR="./lower"
CREATED_MODE="777"
```

Ezek vezérlik az alapértelmezett létrehozást és jogosultságot.

---

## 11.2 need_root()

Ez ellenőrzi, hogy rootként fut-e a script.

Ha nem:

```text
sudo -v
exec sudo -- "$0" "$@"
```

Tehát a script önmagát indítja újra sudo-val.

---

## 11.3 prepare_env()

Ez készíti elő a projektmappát.

Feladatai:

```text
1. ellenőrzi, hogy a projektmappa létezik
2. ellenőrzi vagy létrehozza a hierarchia.txt fájlt
3. létrehozza, ha hiányzik:
   overlay/
   overlay.work/
   out/
4. chmod 777 minden létrehozott elemre
5. ellenőrzi, hogy nincs ':' karakter az overlay útvonalakban
```

---

## 11.4 ensure_hierarchy_file()

Ez ellenőrzi, hogy van-e `hierarchia.txt`.

Ha nincs, kérdez:

```text
Létre akarod hozni az alapértelmezett hierarchia.txt fájlt? [i/N]:
```

Nem válasz esetén kilép.

Igen válasz esetén meghívja:

```text
create_default_hierarchy()
```

---

## 11.5 create_default_hierarchy()

Ez hozza létre:

```text
lower/
hierarchia.txt
```

Sorrend:

```text
1. lower mappa
2. chmod 777 lower
3. hierarchia.txt
4. chmod 777 hierarchia.txt
```

A txt csak akkor készül el, ha a lower mappa létrejött vagy már létezett.

---

## 11.6 ensure_dir_777()

Ez hoz létre mappákat `777` jogosultsággal.

Használja ezekhez:

```text
overlay/
overlay.work/
out/
```

Ha az útvonal symlink, hibát ad, mert ezeknél nem akarunk symlinket.

---

## 11.7 build_lowerdir_string()

Ez olvassa be a `hierarchia.txt` fájlt, és ebből építi fel az overlayfs lowerdir paraméterét.

Feladatai:

```text
1. soronként olvas
2. üres sorokat kihagy
3. kommentelt sorokat kihagy
4. relatív utakat projektmappához képest értelmez
5. readlink -f segítségével felold
6. ellenőrzi, hogy létező mappa-e
7. ellenőrzi, hogy nincs ':' karakter
8. fordított sorrendben összerakja lowerdir stringgé
```

Példa:

```text
hierarchia.txt:
./lower
LAYERS/V101
LAYERS/V102
```

Ebből:

```text
lowerdir=/projekt/LAYERS/V102:/projekt/LAYERS/V101:/projekt/lower
```

---

## 11.8 mount_overlay()

Ez végzi a tényleges mountolást.

Lépések:

```text
1. prepare_env
2. ellenőrzi, hogy out nincs-e már mountolva
3. üríti overlay.work tartalmát, ha maradt benne valami
4. build_lowerdir_string
5. mount -t overlay overlay -o ...
```

A tényleges parancs logikája:

```bash
mount -t overlay overlay \
    -o "lowerdir=${lowerdir_string},upperdir=${upper_dir},workdir=${work_dir}" \
    "$out_dir"
```

---

## 11.9 umount_overlay()

Ez lecsatolja az `out/` mappát, ha mountolva van.

Ha nincs mountolva, nem hibázik, csak kiírja.

---

## 11.10 remount_overlay()

Ez a leggyakoribb művelet.

Lépések:

```text
1. prepare_env
2. ha out mountolva van, umount
3. mount_overlay
```

Ez biztosítja, hogy a módosított `hierarchia.txt` alapján újraépüljön az overlay.

---

# 12. A két program együttműködése

## 12.1 Felelősségek szétválasztása

A rendszerben tiszta felelősségmegosztás van:

```text
dsOverlayGUI:
    kényelmes szerkesztőfelület
    rétegek sorrendezése
    rétegek ki/be kapcsolása
    hierarchia.txt mentése
    dsOverlay meghívása
    mount állapot kijelzése

dsOverlay:
    overlayfs mount/remount/umount
    root jogosultság
    projektmappa előkészítése
    lowerdir string felépítése
    tényleges kernel mount művelet
```

Ez jó felépítés, mert a GUI nem tartalmazza közvetlenül az overlayfs mount logikát. A kritikus rendszerműveletek egy külön, konzolból is használható eszközben vannak.

---

## 12.2 Tipikus munkafolyamat

Egy teljes használati folyamat:

```text
1. dsOverlayGUI indítása
2. projektmappa kiválasztása
3. ha nincs hierarchia.txt, létrehozás
4. rétegek hozzáadása
5. rétegek sorrendjének beállítása
6. szükségtelen rétegek kikapcsolása
7. Mentés
8. Remount
9. out/ mappa használata
```

Ha kézzel akarjuk ugyanezt:

```bash
nano /Data500/projekt1/hierarchia.txt
dsOverlay remount /Data500/projekt1
```

---

# 13. Telepítés

## 13.1 dsOverlay CLI telepítése

Példa:

```bash
sudo cp dsOverlay /opt/bin/dsOverlay
sudo chmod +x /opt/bin/dsOverlay
```

Ellenőrzés:

```bash
/opt/bin/dsOverlay
```

Ha `/opt/bin` benne van a PATH-ban:

```bash
dsOverlay
```

---

## 13.2 dsOverlayGUI telepítése

Példa:

```bash
sudo cp dsOverlayGUI /opt/bin/dsOverlayGUI
sudo chmod +x /opt/bin/dsOverlayGUI
```

Indítás:

```bash
dsOverlayGUI
```

vagy:

```bash
dsOverlayGUI /Data500/projekt1
```

---

## 13.3 Szükséges csomagok

A GUI-hoz szükséges Perl/GTK/WebKit csomagok Debian / Ubuntu / Mint jellegű rendszeren:

```bash
sudo apt install libgtk3-perl libglib-object-introspection-perl libgtk3-webkit2-perl
```

A `pkexec` működéséhez általában kell a polkit:

```bash
sudo apt install policykit-1
```

X11-es root GUI engedélyezéshez szükséges lehet:

```bash
sudo apt install x11-xserver-utils
```

A terminálos futtatáshoz valamelyik terminál:

```bash
sudo apt install xterm
```

Vagy a rendszer saját alap terminálja is elég lehet, ha van `x-terminal-emulator`.

---

# 14. Gyakori hibák és megoldások

## 14.1 Nincs hierarchia.txt

CLI esetén a program rákérdez, létrehozza-e.

GUI esetén figyelmeztető ablak jelenik meg, és a felhasználó dönthet.

Ha nem sikerül létrehozni, ellenőrizni kell a projektmappa jogosultságait.

---

## 14.2 A lower mappa nem jött létre

A program szándékosan nem hozza létre a `hierarchia.txt` fájlt, ha a lower mappa létrehozása elbukik.

Ez helyes működés.

Megoldás:

```bash
ls -ld /projekt/mappa
```

majd jogosultság javítása, vagy rootként indítás.

---

## 14.3 Az out már mountolva van

Ha ezt kapod:

```text
Az out már mountolva van. Használd inkább: remount
```

akkor:

```bash
dsOverlay remount /projekt/mappa
```

---

## 14.4 Nem tűnik el a GTK fájlválasztó / kérdezőablak

Erre a GUI-ban a `flush_gtk_events()` megoldás került be.

Minden kritikus dialog bezárás után:

```text
hide
destroy
flush GTK események
```

---

## 14.5 pkexec után nem nyílik meg a GUI

Lehetséges okok:

```text
DISPLAY nincs átadva
XAUTHORITY nincs átadva
Wayland/X11 jogosultság gond
xhost hiányzik
pkexec policy probléma
```

A program megpróbálja kezelni ezeket, de ha nem sikerül, normál joggal folytatódik.

---

## 14.6 A mount állapot nem azonnal frissül

A GUI minden `Remount` és `Umount` után meghívja a `refresh_web()` függvényt, amely újra lefuttatja a mount ellenőrzést.

Ha külső terminálból kézzel mountolsz vagy umountolsz, akkor a GUI csak a következő saját frissítéskor fogja látni az új állapotot.

---

# 15. Rövid technikai összefoglaló

A rendszer technikailag így néz ki:

```text
Linux kernel overlayfs
    tényleges union filesystem

Bash dsOverlay
    overlayfs mount/remount/umount kezelő

Perl dsOverlayGUI
    fő programlogika és fájlkezelés

GTK3
    natív ablakok, fájlválasztók, message dialogok

WebKit2GTK
    beágyazott HTML/JS felület

HTML/CSS/JS
    látható kezelőfelület, lista, gombok, billentyűk

JSON
    állapot és parancsüzenetek kódolása

document.title trükk
    JS → Perl üzenetküldés

run_javascript()
    Perl → JS állapotfrissítés

pkexec
    rootként történő GUI újraindítás

sudo
    konzolos dsOverlay root jogosultságának biztosítása

mountpoint
    overlay mount állapot ellenőrzése
```

---

# 16. Összegzés

A `dsOverlay` és `dsOverlayGUI` együtt egy praktikus rétegkezelő rendszert alkot.

A konzolos `dsOverlay` önmagában is használható, egyszerű, scriptelhető, és közvetlenül a Linux overlayfs-re épül.

A `dsOverlayGUI` kényelmes grafikus kezelőfelületet ad a `hierarchia.txt` szerkesztéséhez. A főablak HTML/JS alapú, ezért könnyen alakítható, de a tényleges fájl- és rendszerműveletek Perl/GTK/Bash oldalon maradnak.

A két program szétválasztása jó irány, mert:

```text
a CLI egyszerű és megbízható marad
a GUI kényelmesen kezelhető
a kritikus mount logika egy helyen van
a hierarchia.txt kézzel is szerkeszthető
a GUI nem zárja be a rendszert egyetlen használati módba
```

A rendszer így alkalmas mod-rétegek, adatrétegek, verziózott mappák, játékfájlok vagy egyéb overlay alapú munkakörnyezetek kezelésére.

