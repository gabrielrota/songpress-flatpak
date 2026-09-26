
Porting Flatpak di Songpress
1. Preparazione ambiente isolato

flatpak install flathub org.flatpak.Builder
flatpak install flathub org.gnome.Sdk//49
flatpak install flathub org.gnome.Platform//49


Verifica:
flatpak run org.flatpak.Builder --version

Output ottenuto:
flatpak-builder-1.4.9

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
Dalla directory del progetto:
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
flatpak-builder --force-clean --repo=repo build-dir io.github.lallulli.Songpress.yml

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

### Opzione B: Rientra nel container `org.flatpak.Builder` *solo* per la build

Se preferisci non installare nulla con `sudo`, puoi fare la build dentro il container Flatpak e poi eseguire l'app dal tuo sistema host.

1. **Rientra nel container:**
```bash
flatpak run --command=bash org.flatpak.Builder

```


2. **Spostati nella cartella ed esegui la build/installazione:**
```bash
cd ~/songpress-flatpak
flatpak-builder --disable-rofiles-fuse --force-clean --user --install build-dir io.github.lallulli.Songpress.yml

```


3. **Esci dal container:**
```bash
exit

```


4. **Avvia l'app direttamente dal tuo sistema:**
```bash
flatpak run io.github.lallulli.Songpress

```

















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

Rimani dentro il container `org.flatpak.Builder` (oppure sul tuo host se hai installato `flatpak-builder`) ed esegui la build pulita:

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


