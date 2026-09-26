# Songpress Flatpak

Manifest Flatpak comunitario per Songpress, pensato per la revisione dei maintainer upstream e per una possibile pubblicazione su Flathub.

## Stato

- App ID: `io.github.lallulli.Songpress`
- Versione del pacchetto Songpress: `1.9.0` da PyPI
- Runtime e SDK: Freedesktop `25.08`
- Licenza Songpress: GPL-2.0-only
- Build verificata su Linux con `flatpak-builder`

## File del progetto

- `io.github.lallulli.Songpress.yml`: manifest Flatpak
- `python3-songpress.json`: Songpress, pybind11 e dipendenze Python con sorgenti e hash
- `io.github.lallulli.Songpress.desktop`: launcher
- `io.github.lallulli.Songpress.metainfo.xml`: metadati AppStream
- `songpress.png`: icona installata dall'app

Le directory di build, la repository OSTree e i bundle `.flatpak` sono artefatti locali e sono esclusi da Git.

## Prerequisiti

Installare `flatpak-builder` sull'host e il runtime/SDK richiesti dal manifest:

```bash
sudo apt update
sudo apt install flatpak-builder
flatpak install flathub org.freedesktop.Platform//25.08 org.freedesktop.Sdk//25.08
flatpak install flathub org.flatpak.Builder
flatpak-builder --version
```

I comandi di build e installazione vanno lanciati dall'host, nella directory del progetto. L'app Flatpak `org.flatpak.Builder` serve per gli strumenti isolati, ad esempio `flatpak-pip-generator`.

## Dipendenze Python

`python3-songpress.json` è il manifest combinato generato da `flatpak-pip-generator`; contiene già sia `pybind11` sia `songpress`. I due manifest separati non servono.

Per rigenerarlo dopo un aggiornamento delle dipendenze:

```bash
flatpak run --command=flatpak-pip-generator org.flatpak.Builder \
  --output=python3-songpress.json pybind11 songpress
```

Rivedere il diff generato prima di includerlo: la rigenerazione può aggiornare versioni e hash delle dipendenze.

## Validazione e build

Eseguire questi controlli dall'host:

```bash
appstreamcli validate io.github.lallulli.Songpress.metainfo.xml
desktop-file-validate io.github.lallulli.Songpress.desktop
flatpak-builder --disable-rofiles-fuse --force-clean \
  --repo=repo build-dir io.github.lallulli.Songpress.yml
test -s build-dir/files/share/licenses/io.github.lallulli.Songpress/license.txt
```

`--disable-rofiles-fuse` evita l'errore di FUSE riscontrato nell'ambiente di build usato per questo progetto. La build verifica anche che il file di licenza upstream sia installato nell'app.

Per installare localmente e provare l'app:

```bash
flatpak-builder --disable-rofiles-fuse --force-clean --user --install \
  build-dir io.github.lallulli.Songpress.yml
flatpak run io.github.lallulli.Songpress
```

Dopo una modifica al metainfo, rieseguire `appstreamcli validate` e la build. Dopo una modifica al launcher, rieseguire `desktop-file-validate` e la build.

Il metainfo usa uno screenshot della documentazione upstream, con URL fissato a un commit per evitare che l'immagine cambi senza aggiornare il pacchetto.

## Licenze

Songpress `1.9.0` dichiara GNU GPL v2. Il metainfo usa l'identificatore SPDX `GPL-2.0-only`; il file `license.txt` è scaricato dal tag upstream `1.9.0` con SHA-256 fissato nel manifest e installato sotto `/app/share/licenses/`. I pacchetti Python inclusi mantengono i rispettivi avvisi di licenza forniti a monte. La licenza dei metadati AppStream è CC0-1.0.

## Bundle locale

Dopo la build con `--repo=repo`, creare il bundle:

```bash
flatpak build-bundle repo Songpress.flatpak io.github.lallulli.Songpress
flatpak install --user Songpress.flatpak
```

Ricreare il bundle dopo ogni nuova build prima di pubblicarlo. Non committare `repo/`, `build-dir/`, `.flatpak-builder/` o `*.flatpak`.