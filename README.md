
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

