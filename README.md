# TermuxOnePlus

Programas reales de Termux (nano, Python, bash, neofetch…) funcionando dentro de **TermOne Plus**, en Android 15 y sin root.

> Proyecto independiente. No tiene relación con Termux ni con TermOne Plus.

La idea no es solo ejecutar scripts: es **adaptar (portar)** programas y entender qué permite y qué impide el sistema.

## El truco en una línea

```sh
/system/bin/linker64 ~/.apt/real/bin/nano
```

Android 15 no deja ejecutar archivos copiados a la carpeta de una app, pero el cargador del sistema (`linker64`) sí puede abrir un binario aunque no tenga permiso de ejecución. Con eso, más las librerías empaquetadas y `LD_LIBRARY_PATH`, el programa corre.

## Qué se descubrió

| Hallazgo | Consecuencia |
|---|---|
| Los paquetes de Termux llevan fija la ruta `/data/data/com.termux/files/usr` | No se pueden instalar tal cual |
| Ejecutar un binario copiado a la carpeta de TermOne da `Permission denied` | No sirve `chmod +x` ni `./programa` |
| `/system/bin/linker64 binario` sí lo ejecuta | Vía principal del proyecto |
| El shell de TermOne solo trae `toybox` (sin `curl`, `wget`, `ar`, `xzcat`; sí `tar`, `gzip`, `nc`, `base64`) | No hay descargas desde internet |
| TermOne no puede **leer** archivos que Termux deja en `/sdcard/Download` | El paquete viaja por `nc` en localhost |
| En Termux puede faltar `readelf` | Las dependencias se leen del binario con `grep` |

## Estado

**Probado y funcionando**

- GNU nano 9.3
- Python 3.14.10
- bash 5.3.20
- neofetch
- En sh puro: `apt.sh`, `nano.sh`, `py.sh`, `ahorcado.sh`, `calc`, `adivina`, `sysinfo`, `cos`

**Por probar**

- `wget` (con certificados de Termux)
- `lynx` (navegador CLI; pide un `lynx.lss` que se crea vacío)
- `fastfetch`
- `nInvaders` (juego)
- `mpv` con `--vo=tct` (video en terminal)

## Cómo pasar un programa de Termux a TermOne

### 1. En Termux

Una sola vez: `termux-setup-storage` (acepta el permiso). Luego, con `mkmulti.sh`:

```sh
sh mkmulti.sh nano nano                          # un programa
EXTRA="etc/lynx.cfg" sh mkmulti.sh extras bash neofetch lynx   # varios, con archivos extra
```

Deja `NOMBRE-real.tgz` en `/sdcard/Download` con el binario, sus librerías, `terminfo` y los extra.

### 2. Enviarlo por la red local

```sh
# TermOne (queda esperando)
nc -l -p 8888 > ~/nano-real.tgz
# Termux (otra sesión)
nc -N localhost 8888 < /sdcard/Download/nano-real.tgz
# TermOne
mkdir -p ~/.apt/real && tar xzf ~/nano-real.tgz -C ~/.apt/real
```

### 3. Lanzador en `~/.apt/real/env.sh`

```sh
export TERM=${TERM:-xterm-256color}
R=$HOME/.apt/real
rexec() { b=$1; shift; LD_LIBRARY_PATH=$R/lib TERMINFO=$R/share/terminfo PYTHONHOME=$R /system/bin/linker64 $R/bin/$b "$@"; }
nano() { rexec nano "$@"; }
python() { rexec python3 "$@"; }
bash() { rexec bash --norc --noprofile "$@"; }
neofetch() { rexec bash --norc --noprofile $R/bin/neofetch "$@"; }
export PYTHON_BASIC_REPL=1
```

Se activa con `. ~/.apt/real/env.sh` (conviene ponerlo en el comando de inicio de TermOne).

### Python

Va igual, pero hay que copiar también la librería estándar (`$PREFIX/lib/python3.x`) y las `.so` de `lib-dynload`. Se excluyen `test`, `idlelib` y `pip` para ahorrar espacio.

## Versión en sh puro (sin binarios)

Funciona en cualquier shell de Android, sin permisos especiales.

| Archivo | Qué es |
|---|---|
| `nano.sh` | Editor por líneas: añadir, editar, insertar, borrar, buscar, guardar |
| `py.sh` | Mini Python: `print`, variables, aritmética, `if/elif/else`, `while`, `for range`, `break`, `input` |
| `apt.sh` | Gestor de paquetes sobre un repo en carpeta local: `setup`, `set-repo`, `update`, `install`, `remove`, `list`, `search`, `show`, `upgrade`, `mkrepo` |
| `paquetes.sh` | Genera `repo/` con `sysinfo`, `calc`, `adivina`, `cos`, `nano` y `py` |
| `ahorcado.sh` | Juego del ahorcado |

Índice `Packages` del repo (una línea por paquete): `nombre|version|archivo.sh|descripcion`

```sh
sh paquetes.sh && sh apt.sh setup && . ~/.apt/env.sh
apt set-repo $PWD/repo && apt update
apt install sysinfo calc adivina cos nano py
```

## Scripts de Termux

| Archivo | Uso |
|---|---|
| `mkmulti.sh` | Empaqueta varios programas en un `.tgz` (recomendado) |
| `mkbundle.sh` | Empaqueta un programa; también sirve para Python (`sh mkbundle.sh python python3`) |

Las librerías se detectan leyendo los nombres `lib*.so*` del binario (con `readelf` si existe, y con `grep` si no) y se copian de forma recursiva desde `$PREFIX/lib`.

## Errores comunes

| Mensaje | Causa y arreglo |
|---|---|
| `Cannot open: Operation not permitted` al crear el `.tgz` | Falta el permiso de almacenamiento en Termux: `termux-setup-storage` |
| `tar: ... Permission denied` en TermOne | TermOne no lee archivos de Termux: enviar por `nc` |
| `library "libX.so" not found` | Falta una librería en `~/.apt/real/lib`: volver a empaquetar |
| `unable to open file .../bin/nano` | El `.tgz` no se extrajo en `~/.apt/real` |
| `Error opening terminal` | Falta `TERM`: `export TERM=xterm-256color` |
| `warning: can't use pyrepl` | Cosmético: `export PYTHON_BASIC_REPL=1` |

## Límites

- Probado en Android 15. Otras versiones pueden comportarse distinto.
- `apt.sh` solo lee repos en carpeta local, mientras no haya `wget` funcionando en TermOne.
- `py.sh` no tiene funciones, listas ni concatenación de cadenas.
- Algunos programas llevan rutas fijas de Termux dentro del binario y pueden necesitar ajustes (variables de entorno o archivos extra).

## Hoja de ruta

- [x] Editor, mini Python y gestor de paquetes en sh
- [x] nano, Python, bash y neofetch reales con `linker64`
- [ ] Probar `wget`, `lynx`, `fastfetch`, `nInvaders` y `mpv`
- [ ] `apt install-real`: instalar un `.tgz` y registrar el comando solo
- [ ] Conectar `wget` real con `apt.sh` para instalar desde GitHub Pages
- [ ] Portar proyectos propios (Pluma, ClaudeOS) a sh
