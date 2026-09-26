
Porting Flatpak di Songpress
1. Preparazione ambiente isolato

flatpak install flathub org.flatpak.Builder
flatpak install flathub org.gnome.Sdk//49
flatpak install flathub org.gnome.Platform//49


Verifica:
flatpak run org.flatpak.Builder --version

Output ottenuto:
flatpak-builder-1.4.9

## Scelta dell'ambiente di build

Per build, installazione locale ed esportazione del bundle, usare `flatpak-builder` installato sull'host:

```bash
sudo apt update
sudo apt install flatpak-builder
flatpak-builder --version
```

Eseguire questi comandi dal terminale host, nella directory del progetto. In questo modo `flatpak-builder` può integrarsi direttamente con Flatpak dell'utente per `--user --install` e per creare un repository di rilascio.

Usare invece `org.flatpak.Builder` per gli strumenti isolati, ad esempio `flatpak-pip-generator`. Non è necessario entrare nella sua shell prima di lanciare i comandi di build sull'host.

2. Accesso al runtime di build
flatpak run --command=bash org.flatpak.Builder

Verifiche effettuate:

python3 --version
pip3 --version
cat /etc/os-release


Risultato:

Python 3.13.15
pip 26.2.1
Freedesktop SDK 25.08


Questo conferma che si sta lavorando in un ambiente isolato Flatpak e non sul sistema host Ubuntu.
3. Generazione dipendenze Python
Verifica presenza del generatore:
which flatpak-pip-generator

Risultato:
/app/bin/flatpak-pip-generator

Generazione modulo:
flatpak-pip-generator songpress
flatpak-pip-generator pybind11

File generato:
python3-songpress.json

4. Verifica della dipendenza critica wxPython
grep -i wx python3-songpress.json

Risultato:
wxpython-4.3.1.tar.gz

Conclusioni:

- Songpress dichiara correttamente wxPython.
- flatpak-pip-generator lo ha risolto automaticamente.
- wxPython verrà compilato durante la build Flatpak.
- Il prossimo ostacolo probabile saranno eventuali librerie native richieste da wxPython.

5. Creazione directory di lavoro
Sull'host:

mkdir -p ~/songpress-flatpak
cd ~/songpress-flatpak


Copiare qui:
python3-songpress.json

Struttura prevista:

songpress-flatpak/
├── io.github.lallulli.Songpress.yml
└── python3-songpress.json


6. Manifest Flatpak minimale
Creare il file:
io.github.lallulli.Songpress.yml

Contenuto:

app-id: io.github.lallulli.Songpress

runtime: org.freedesktop.Platform
runtime-version: "25.08"

sdk: org.freedesktop.Sdk

command: songpress

finish-args:
  - --share=ipc
  - --socket=fallback-x11
  - --socket=wayland
  - --filesystem=home

modules:
  - python3-pybind11.json
  - python3-songpress.json


7. Primo tentativo di build
Dalla directory del progetto, in un terminale host:
flatpak-builder --disable-rofiles-fuse --force-clean build-dir io.github.lallulli.Songpress.yml

Possibili esiti:

1. La build completa con successo.
2. La build si interrompe durante la compilazione di wxPython.
3. Mancano librerie native nel runtime SDK e dovranno essere aggiunte al manifest.
8. Comandi diagnostici utili
head -40 python3-songpress.json


grep -i wx python3-songpress.json


grep -i songpress python3-songpress.json

tail -100 .flatpak-builder/last-build.log

Stato attuale
✅ Runtime Flatpak funzionante ✅ Python disponibile nel sandbox ✅ pip disponibile nel sandbox ✅ flatpak-pip-generator disponibile ✅ python3-songpress.json generato ✅ wxPython individuato tra le dipendenze ⏳ Primo tentativo di build del manifest



# TODO

- [] 2026-07-26 10:18 - la build è troppo pesante perché ricompila tutto pybind11 va provato quando consigliato da Gemini
``` bash
flatpak run --command=bash org.flatpak.Builder
flatpak-pip-generator pybind11 songpress
exit

```



# test
Ora che la build è andata a buon fine e la directory `build-dir` è stata popolata, le fasi successive sono tre: **testare** l'applicazione, **installarla** localmente (o crearne il pacchetto) e infine **verificare** che si avvii correttamente.

Ecco i comandi da eseguire in sequenza:

---

### 1. Testare l'app direttamente dalla directory di build

Puoi eseguire l'applicazione immediatamente all'interno dell'ambiente Flatpak per verificare che tutte le dipendenze Python/C++ funzionino senza dover ancora installare nulla nel sistema:

```bash
flatpak-builder --run build-dir io.github.lallulli.Songpress.yml songpress

fusermount: file descriptor 4 is not a socket, can't send fuse fd
Error running songpress: Failure spawning rofiles-fuse, exit_status: 256

```

*(Se l'interfaccia grafica si apre ed è funzionante, la build è perfetta).*

---

### 2. Installare l'app nel tuo ambiente Flatpak utente

Per fare in modo che l'applicazione sia installata sul sistema (disponibile nel menu delle applicazioni dell'ambiente grafico), aggiungi il flag `--user --install`:

```bash
flatpak-builder --user --install --force-clean build-dir io.github.lallulli.Songpress.yml

```

A questo punto potrai avviarla semplicemente con:

```bash
flatpak run io.github.lallulli.Songpress

```

---

### 3. (Opzionale) Esportare un pacchetto bundle `.flatpak`

Se vuoi distribuire l'applicazione su altre macchine o condividerla, puoi generare un unico file `.flatpak` creando prima un repository locale (ostree repo) e poi esportando il bundle:

```bash
# 1. Esporta la build in un repository temporaneo
flatpak-builder --disable-rofiles-fuse --force-clean --repo=repo build-dir io.github.lallulli.Songpress.yml

# 2. Crea il file .flatpak pronto da distribuire
flatpak build-bundle repo Songpress.flatpak io.github.lallulli.Songpress

```

---








Il problema è dovuto a **`rofiles-fuse`**, un meccanismo di sicurezza usato da Flatpak che in alcuni ambienti (come container, macchine virtuali o ambienti senza permessi FUSE) non riesce ad agganciare il socket del filesystem.

L'errore è derivato dal fatto che avevi usato `--disable-rofiles-fuse` durante la fase di build, ma il comando `flatpak-builder --run` cerca comunque di avviarlo di default.

Per risolvere, hai due alternative:

---

### Opzione 1: Disabilitare FUSE anche sul comando `--run` (metodo più rapido)

Aggiungi il flag `--disable-rofiles-fuse` anche al comando di esecuzione:

```bash
flatpak-builder --disable-rofiles-fuse --run build-dir io.github.lallulli.Songpress.yml songpress

```

---

### Opzione 2: Installare l'app direttamente ed eseguirla con `flatpak run`

Invece di usare `flatpak-builder --run`, installa il risultato della build nel tuo ambiente utente e avviala normalmente via `flatpak`:

```bash
# 1. Installa la build corrente senza usare rofiles-fuse
flatpak-builder --disable-rofiles-fuse --user --install build-dir io.github.lallulli.Songpress.yml

# 2. Avvia l'applicazione installata
flatpak run io.github.lallulli.Songpress

```

Questo aggira completamente la necessità di `rofiles-fuse` e ti permette anche di testare l'integrazione nel sistema (come l'icona nel menu o i file `.desktop`).














L'opzione `--disable-rofiles-fuse` è valida soltanto per il comando di **build** di `flatpak-builder`, ma non esiste per la modalità `--run`.

Per avviare l'applicazione evitando i problemi di FUSE, la soluzione corretta è **installare direttamente la build** nel tuo profilo Flatpak utente e poi eseguirla con `flatpak run`.

Esegui questi due comandi:

```bash
# 1. Installa l'applicazione compilata
flatpak-builder --disable-rofiles-fuse --user --install build-dir io.github.lallulli.Songpress.yml

# 2. Avvia l'applicazione
flatpak run io.github.lallulli.Songpress

```

---

### In alternativa: entrare nella sandbox senza FUSE

Se vuoi comunque esplorare l'ambiente di runtime prima dell'installazione, puoi usare `flatpak build` puntando direttamente alla cartella `build-dir`:

```bash
flatpak build build-dir songpress

```









Il messaggio ti avvisa che la cartella `build-dir` contiene già i file della build precedente e che `flatpak-builder` non vuole sovrascriverla senza permessi.

Per risolvere, aggiungi il flag **`--force-clean`** per pulire la cartella prima dell'installazione:

```bash
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml

```

Una volta terminata l'installazione, avvia l'app con:

```bash
flatpak run io.github.lallulli.Songpress

```
















**Sì, esattamente.** Il motivo dell'errore è proprio che stai lanciando `flatpak run` dall'interno della sandbox di `org.flatpak.Builder`.

`bwrap` (Bubblewrap), la tecnologia alla base di Flatpak per creare i container, ha bisogno di creare **namespace utente non privilegiati** (`user namespaces`). Quando provi ad avviare un Flatpak dentro un altro Flatpak (un Flatpak "nested"), la sandbox madre blocca la creazione di un secondo namespace per motivi di sicurezza, restituendo l'errore:

```text
bwrap: No permissions to create a new namespace...

```

---

### Come risolvere

#### 1. Esci dalla shell del container

Per prima cosa, esci dall'ambiente di `org.flatpak.Builder`:

```bash
exit

```

#### 2. Esegui il comando dalla shell del tuo sistema (host)

Torna nella cartella del progetto sul tuo terminale locale:

```bash
cd ~/songpress-flatpak
flatpak run io.github.lallulli.Songpress

```

---

### Nota sui percorsi XDG_DATA_DIRS

Se dopo esser uscito dall'ambiente `org.flatpak.Builder` l'app parte ma l'icona non appare ancora nel menu delle applicazioni della tua distribuzione, aggiorna le variabili di ambiente per la sessione corrente con:

```bash
export XDG_DATA_DIRS="$XDG_DATA_DIRS:$HOME/.local/share/flatpak/exports/share:/var/lib/flatpak/exports/share"

```

In alternativa, disconnetti e riconnetti l'utente oppure riavvia la sessione grafica per aggiornare l'elenco delle applicazioni di sistema.





















L'errore `Unable to access the X Display, is $DISPLAY set properly?` indica che l'applicazione sta cercando di aprire un'interfaccia grafica (GUI), ma la sandbox di Flatpak **non ha i permessi di accesso al server grafico** del tuo sistema (X11 / Wayland) oppure l'ambiente non ha la variabile `$DISPLAY` configurata.

Per permettere a Songpress di visualizzare la finestra di dialogo e l'interfaccia utente, devi concedere i permessi di rete e visualizzazione nel manifest YAML.

---

### Soluzione: Aggiungi i permessi di socket nel manifest

Apri il file `io.github.lallulli.Songpress.yml` e assicurati che nella sezione `finish-args` siano presenti i socket per X11 e Wayland (oltre a eventuali permessi di rete se necessari):

```yaml
app-id: io.github.lallulli.Songpress
runtime: org.freedesktop.Platform
runtime-version: "25.08"
sdk: org.freedesktop.Sdk
command: songpress

finish-args:
  # Permessi per l'interfaccia grafica (X11 e Wayland)
  - --socket=x11
  - --socket=fallback-x11
  - --socket=wayland
  # Abilita l'accelerazione grafica (se necessaria)
  - --device=dri
  # Permette di mostrare le notifiche di sistema (opzionale)
  - --talk-name=org.freedesktop.Notifications

modules:
  - python3-pybind11.json
  - python3-songpress.json

```

---

### Prossimi passaggi:

1. **Reinstalla l'app** per applicare i nuovi permessi del manifest:
```bash
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml

```


2. **Lancia nuovamente l'applicazione**:
```bash
flatpak run io.github.lallulli.Songpress

```



*(Nota: se lanci il comando via SSH o da una sessione terminale puramente testuale senza server grafico attivo, assicurati di eseguire il comando da un terminale avviato all'interno del tuo ambiente desktop, ad es. GNOME Terminal o Konsole).*






















Questo messaggio indica semplicemente che **sei uscito dal container** `org.flatpak.Builder` (come avevamo visto al passaggio precedente) e ora ti trovi sulla tua macchina reale (l'host), dove il pacchetto `flatpak-builder` non è ancora installato a livello di sistema operativo.

Hai **due opzioni** per proseguire:

---

### Opzione A: Installa `flatpak-builder` sul tuo sistema (Consigliato)

È la soluzione più comoda e pulita se intendi sviluppare o compilare Flatpak in futuro.

Installa il pacchetto sulla tua Ubuntu/Debian eseguendo:

```bash
sudo apt update && sudo apt install flatpak-builder -y

```

Dopo averlo installato, compila/installa di nuovo l'app con i nuovi permessi grafici nel manifest:

```bash
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml

```

E infine avviala con:

```bash
flatpak run io.github.lallulli.Songpress

```

---

### Opzione B: usare il container `org.flatpak.Builder`

Il container resta utile per eseguire strumenti isolati come `flatpak-pip-generator`. Per build, installazione e creazione del bundle, seguire invece l'opzione A ed eseguire `flatpak-builder` dall'host: evita problemi di integrazione con Flatpak e con l'installazione utente.

Per generare o aggiornare le dipendenze Python, aprire la shell isolata:

```bash
flatpak run --command=bash org.flatpak.Builder
```

Per compilare e installare l'app, uscire dalla shell con `exit` e usare i comandi dell'opzione A dal terminale host.

















Ottimo che l'applicazione ora funzioni!

Ora che hai generato `python3-modules.json` tramite `flatpak-pip-generator`, l'obiettivo è sostituire la compilazione da sorgente con i **pacchetti wheel (.whl) o moduli precompilati**, velocizzando enormemente le build successive.

Ecco come integrare il nuovo file ed eliminare le parti lente:

---

### 1. Sostituisci i moduli nel file `io.github.lallulli.Songpress.yml`

Apri il file manifest `io.github.lallulli.Songpress.yml` e aggiorna la sezione `modules`:

```yaml
app-id: io.github.lallulli.Songpress
runtime: org.freedesktop.Platform
runtime-version: "25.08"
sdk: org.freedesktop.Sdk
command: songpress

finish-args:
  - --socket=x11
  - --socket=fallback-x11
  - --socket=wayland
  - --device=dri
  - --talk-name=org.freedesktop.Notifications

modules:
  # Includi il file generato da flatpak-pip-generator
  - python3-modules.json

```

> **Nota:** Rimuovi `- python3-pybind11.json` e `- python3-songpress.json` e usa solo `- python3-modules.json` (che ora gestisce sia `pybind11` sia `songpress` e le loro dipendenze).

---

### 2. Esegui la nuova build veloce

Dal terminale host, nella directory del progetto, esegui la build pulita:

```bash
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml

```

Noterai che la build sarà **molto più rapida**, poiché `flatpak-builder` non compilerà più i file C++ da zero ma installerà direttamente le dipendenze Python già pronte e scaricate da PyPI.

---

### 3. Verifica il funzionamento

Esci dall'ambiente `org.flatpak.Builder` (se stai usando il container):

```bash
exit

```

E lancia l'applicazione per verificare che si avvii correttamente con i nuovi pacchetti installati:

```bash
flatpak run io.github.lallulli.Songpress

```






















**Sì, è assolutamente previsto**, a patto che l'applicazione fornisca un file `.desktop` valido e le relative icone nei percorsi standard di Flatpak.

In Flatpak, l'icona nel launcher/menu di sistema del tuo ambiente desktop (GNOME, KDE, XFCE, ecc.) appare se vengono soddisfatte queste due condizioni nel file manifest o nella struttura di installazione:

---

### 1. I requisiti per far apparire l'icona

Per far sì che l'ambiente grafico riconosca l'applicazione e mostri l'icona, nella directory di destinazione dell'app (`/app/share/` all'interno della sandbox) devono essere presenti:

1. **Un file `.desktop**` nominato *esattamente* come l'App ID:
* `/app/share/applications/io.github.lallulli.Songpress.desktop`


2. **Un'icona** (in formato PNG o SVG) posizionata nel percorso delle icone standard, anch'essa nominata con l'App ID:
* `/app/share/icons/hicolor/scalable/apps/io.github.lallulli.Songpress.svg` *(oppure in `128x128/apps/io.github.lallulli.Songpress.png`)*


3. **Il riferimento dentro il file `.desktop**`:
All'interno del file `.desktop`, la voce `Icon` deve corrispondere all'App ID:
```ini
[Desktop Entry]
Name=Songpress
Exec=songpress
Icon=io.github.lallulli.Songpress
Type=Application
Categories=AudioVideo;Audio;

```



---

### 2. Perché potresti non vederla subito?

Se la build include già il file `.desktop` e l'icona, ma non la vedi ancora nel menu di sistema:

* **Sessione non aggiornata (XDG_DATA_DIRS)**: Quando installi un pacchetto utente con `--user`, l'ambiente grafico spesso ha bisogno di un riavvio della sessione (logout/login) o dell'aggiornamento manuale del database desktop.
* **Aggiorna la cache del menu manualmente** con questo comando:
```bash
update-desktop-database ~/.local/share/flatpak/exports/share/applications

```



---

### 3. Come aggiungere l'icona se Songpress non la installa da solo

Se durante la build Songpress non copia il file `.desktop` e l'icona nei percorsi corretti, puoi forzarne la creazione direttamente nel tuo manifest YAML aggiungendo una sezione di post-installazione (`build-commands`):

```yaml
  - name: songpress
    buildsystem: simple
    # ... le tue sorgenti ...
    post-install:
      # Installa l'icona
      - install -Dm644 extra/songpress.png /app/share/icons/hicolor/128x128/apps/io.github.lallulli.Songpress.png
      # Installa il file .desktop
      - install -Dm644 extra/songpress.desktop /app/share/applications/io.github.lallulli.Songpress.desktop

```
















Sì, affinché l'icona e il menu di sistema riconoscano correttamente l'applicazione, devi fare **due passaggi**:

1. Assicurarti che il file `.desktop` abbia i campi corretti.
2. Copiare il file `.desktop` e l'icona nei percorsi standard dell'applicazione durante la fase di build.

Ecco la procedura passo-passo:

---

### 1. Verificare il file `io.github.lallulli.Songpress.desktop`

Apri il file `.desktop` che hai creato e assicurati che contenga almeno queste righe fondamentali (in particolare `Exec` e `Icon` devono riflettere l'App ID):

```ini
[Desktop Entry]
Type=Application
Name=Songpress
Comment=Edit and format songs with chords
Exec=songpress
Icon=io.github.lallulli.Songpress
Terminal=false
Categories=AudioVideo;Audio;

```

---

### 2. Copiare `.desktop` e Icona tramite il manifest YAML

Nel tuo file `io.github.lallulli.Songpress.yml`, devi fare in modo che `flatpak-builder` copi il file `.desktop` e un'immagine dell'icona (ad es. `songpress.png` o `songpress.svg`) nelle cartelle di sistema di Flatpak (`/app/share/...`).

Aggiungi (o aggiorna) un modulo dedicato o una sezione `post-install` nel tuo manifest:

```yaml
app-id: io.github.lallulli.Songpress
runtime: org.freedesktop.Platform
runtime-version: "25.08"
sdk: org.freedesktop.Sdk
command: songpress

finish-args:
  - --socket=x11
  - --socket=fallback-x11
  - --socket=wayland
  - --device=dri
  - --talk-name=org.freedesktop.Notifications

modules:
  - python3-modules.json

  # Modulo per installare il file .desktop e l'icona dell'applicazione
  - name: songpress-desktop-integration
    buildsystem: simple
    build-commands:
      # Installa il file .desktop
      - install -Dm644 io.github.lallulli.Songpress.desktop /app/share/applications/io.github.lallulli.Songpress.desktop
      # Installa l'icona (sostituisci "songpress.png" con il percorso/nome reale della tua icona)
      - install -Dm644 songpress.png /app/share/icons/hicolor/128x128/apps/io.github.lallulli.Songpress.png
    sources:
      - type: file
        path: io.github.lallulli.Songpress.desktop
      - type: file
        path: songpress.png # Modifica se l'icona ha un nome/estensione diversa

```

L'immagine l'ho presa da qui ![songpress_icon.png](https://github.com/lallulli/songpress/blob/master/docs/assets/songpress_icon.png)

---

### 3. Ricompilare e verificare

Ora riesegui la build ed installazione:

```bash
flatpak run --command=bash org.flatpak.Builder
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml
exit
```

Infine, aggiorna il database del desktop per far apparire subito l'icona nel launcher (senza dover riavviare la sessione):

```bash
update-desktop-database ~/.local/share/flatpak/exports/share/applications

```

Ora dovresti trovare **Songpress** direttamente nel menu delle applicazioni della tua distribuzione!













Stai procedendo benissimo: il link simbolico è stato creato correttamente e Flatpak ha generato automaticamente il file `.desktop` di esportazione (`Exec=/app/bin/flatpak run ...`).

Il motivo per cui **non compare ancora nel menu** è legato a due piccoli dettagli tecnici che possiamo sistemare subito:

---

### 1. Il problema del comando `Exec` nel file `.desktop` generato

Guardando il contenuto del file tramite il comando `cat`, la riga `Exec` punta a `/app/bin/flatpak run ...`. Quel percorso però si riferisce *all'interno* della sandbox o a un wrapper errato generato da Flatpak se non ha riconosciuto correttamente il binario principale.

Dato che il comando principale del tuo manifest è `command: songpress`, il file `.desktop` deve semplicemente richiamare il comando o lasciare che sia Flatpak a gestirlo pulito.

Modifica il tuo file sorgente `io.github.lallulli.Songpress.desktop` (quello che hai creato tu nella cartella del progetto) impostando la riga `Exec` in questo modo:

```ini
[Desktop Entry]
Type=Application
Name=Songpress
Comment=Edit and format songs with chords
Exec=songpress
Icon=io.github.lallulli.Songpress
Terminal=false
Categories=AudioVideo;Audio;

```

*(Nota: basta inserire `Exec=songpress`, perché Flatpak sa già come instradarlo).*

---

### 2. Controlla che l'icona sia al suo posto

Dall'output di `ll` vedo che il file `.desktop` c'è, ma manca il controllo sull'icona. Affinché il desktop environment (GNOME, KDE, ecc.) mostri l'applicazione nel menu, spesso pretenda che l'icona specificata (`Icon=io.github.lallulli.Songpress`) esista effettivamente nella cartella dei temi (`hicolor`).

Assicurati che nel blocco `post-install` del tuo manifest YAML l'icona venga copiata correttamente:

```yaml
  - name: songpress-desktop-integration
    buildsystem: simple
    build-commands:
      # Installa il file .desktop
      - install -Dm644 io.github.lallulli.Songpress.desktop /app/share/applications/io.github.lallulli.Songpress.desktop
      # Installa l'icona (assicurati di avere un file 'songpress.png' o '.svg' nella cartella del progetto)
      - install -Dm644 songpress.png /app/share/icons/hicolor/128x128/apps/io.github.lallulli.Songpress.png
    sources:
      - type: file
        path: io.github.lallulli.Songpress.desktop
      - type: file
        path: songpress.png

```

---

### 3. Ricrea la build e aggiorna la cache

Dopo aver sistemato il file `.desktop` sorgente e verificato l'icona, riesegui la build e l'installazione:

```bash
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml

```

Infine, forza l'aggiornamento del database delle applicazioni desktop e delle icone sul tuo sistema host:

```bash
update-desktop-database ~/.local/share/flatpak/exports/share/applications
gtk-update-icon-cache -f -t ~/.local/share/icons/hicolor 2>/dev/null || true

```

A questo punto, cercando "Songpress" nel menu delle applicazioni del tuo computer, l'icona dovrebbe finalmente comparire!




il problema era la composizione dell'exec rimuovendo /app/bin/ da /app/bin/flatpak e lasciando solo flatpak è andato a posto
qual'è il prossimo passo per il rilascio flatpak?












Fantastico, ottimo lavoro sul risolvere il path di `Exec`!

Per completare il pacchetto e prepararlo per il **rilascio ufficiale** (ad esempio su **Flathub** o come bundle distribuibile), il percorso standard prevede tre fasi principali:

---

### 1. Aggiungere il file AppStream (`.metainfo.xml`)

Flathub e gli store grafici Linux (come *GNOME Software* e *KDE Discover*) **richiedono obbligatoriamente** un file AppStream che descriva l'applicazione, fornisca gli screenshot e indichi le licenze.

Crea un file chiamato `io.github.lallulli.Songpress.metainfo.xml` nella cartella del tuo progetto con questa struttura base:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<component type="desktop-application">
  <id>io.github.lallulli.Songpress</id>
  <metadata_licence>CC0-1.0</metadata_licence>
  <project_license>GPL-3.0-or-later</project_license>
  <name>Songpress</name>
  <summary>Edit and format songs with chords</summary>
  
  <description>
    <p>
      Songpress è uno strumento per formattare, stampare e gestire canzoni con accordi.
    </p>
  </description>
  
  <launchable type="desktop-id">io.github.lallulli.Songpress.desktop</launchable>
  
  <screenshots>
    <screenshot type="default">
      <caption>Schermata principale di Songpress</caption>
      <image>https://raw.githubusercontent.com/lallulli/songpress/main/extra/screenshot.png</image>
    </screenshot>
  </screenshots>

  <url type="homepage">https://github.com/lallulli/songpress</url>
</component>

```

Installa il file `.metainfo.xml` nel manifest aggiungendo al blocco `post-install` del tuo modulo:

```bash
install -Dm644 io.github.lallulli.Songpress.metainfo.xml /app/share/metainfo/io.github.lallulli.Songpress.metainfo.xml

```

---

### 2. Validare il manifest con `flatpak-builder` ed `appstreamcli`

Prima di inviare l'app, è bene verificare che rispetti le linee guida e gli standard di sicurezza di Flatpak:

```bash
# Validazione del file metainfo AppStream
appstreamcli validate io.github.lallulli.Songpress.metainfo.xml

# Validazione del file .desktop
desktop-file-validate io.github.lallulli.Songpress.desktop

# Build di verifica e controllo che la licenza upstream sia inclusa
flatpak-builder --disable-rofiles-fuse --force-clean --repo=repo build-dir io.github.lallulli.Songpress.yml
test -s build-dir/files/share/licenses/io.github.lallulli.Songpress/license.txt

```

Dopo ogni modifica al metainfo, inclusa la licenza, rieseguire `appstreamcli validate` e la build qui sopra: la build aggiorna i metadati esportati e ricrea il repository. Se si distribuisce un bundle, ricrearlo dopo la build con `flatpak build-bundle repo Songpress.flatpak io.github.lallulli.Songpress`. `desktop-file-validate` va ripetuto quando cambia il file `.desktop`.

---

### 3. Scegliere la modalità di distribuzione

Hai **due strade principali** a seconda di come desideri distribuire l'applicazione:

#### Opzione A: Pubblicare su Flathub (Consigliato)

Flathub è l'app store di riferimento dell'ecosistema Linux.

1. Fai il **fork** del repository GitHub di [Flathub Submission](https://github.com/flathub/flathub?utm_source=gemini).
2. Crea un ramo con i file del tuo manifest (`io.github.lallulli.Songpress.yml`, `python3-modules.json`, il file `.desktop` e il file `.metainfo.xml`).
3. Apri una **Pull Request** verso il repository di Flathub. I maintainer di Flathub eseguiranno dei test automatizzati ed effettueranno una review del manifest per approvare la pubblicazione.

---

#### Opzione B: Generare un bundle standalone (`.flatpak`)

Se desideri distribuire l'applicazione direttamente come file scaricabile da allegare alle *Releases* di GitHub senza passare da uno store:

1. Compila l'applicazione esportandola in un repository locale OSTree:
```bash
flatpak-builder --disable-rofiles-fuse --force-clean --repo=repo build-dir io.github.lallulli.Songpress.yml

```


2. Genera il file unico `.flatpak`:
```bash
flatpak build-bundle repo Songpress.flatpak io.github.lallulli.Songpress

```



In questo modo i tuoi utenti potranno installare l'applicazione con un semplice doppio clic sul file `Songpress.flatpak` o via terminale con:

```bash
flatpak install Songpress.flatpak

```


```bash

appstreamcli validate io.github.lallulli.Songpress.metainfo.xml
flatpak-builder --disable-rofiles-fuse --force-clean --repo=repo build-dir io.github.lallulli.Songpress.yml
test -s build-dir/files/share/licenses/io.github.lallulli.Songpress/license.txt

```






