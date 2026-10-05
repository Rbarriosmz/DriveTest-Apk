# iTEAM / NetMonitorOSM

**Aplicación Android de drive-test celular (estilo G-MoM Pro / QualiPoc).**
Registra en el tiempo la celda servidora y vecinas (4G/5G), el GPS y el throughput, y
opcionalmente ejecuta un *script* de tráfico con iPerf3 (UL/DL/UDP), HTTP y Vídeo QoE, con
ping en paralelo. Todo se guarda en CSV y se visualiza sobre un mapa OpenStreetMap.

- **Paquete:** `com.netmon.osm`  ·  **Nombre:** iTEAM
- **Android:** 7.0+ (API 24+), optimizado para 5G NR (SA y NSA)
- **Autor:** Ricardo Barrios · [rnbarmuo@iteam.upv.es](mailto:rnbarmuo@iteam.upv.es) · Instituto de Telecomunicaciones y Aplicaciones Multimedia (iTEAM) — UPV

---

## Índice
1. [Instalación](#instalación)
2. [Permisos y primera configuración](#permisos-y-primera-configuración)
3. [Las ventanas (pestañas)](#las-ventanas-pestañas)
   - [Mapa](#1-mapa) · [Red](#2-red) · [iPerf3](#3-iperf3) · [Gráfica](#4-gráfica)
4. [Configurar el script](#configurar-el-script)
5. [Ejemplo con servidores iPerf3 públicos](#ejemplo-con-servidores-iperf3-públicos)
6. [Grabación](#grabación)
7. [Control por ADB](#control-por-adb-opcional)
8. [Archivos de salida (reportes)](#archivos-de-salida-reportes)
9. [Menú](#menú-tres-puntos)
10. [Limitaciones conocidas](#limitaciones-conocidas)

---

## Instalación

```bash
adb -s <serie> install -r app/build/outputs/apk/debug/app-debug.apk
```

Recompilar (opcional):

```bash
# Ubuntu
export JAVA_HOME=~/android-studio/jbr
./gradlew :app:assembleDebug

# Windows
set JAVA_HOME=C:\Program Files\Android\Android Studio\jbr
gradlew.bat :app:assembleDebug
```

El APK queda en `app/build/outputs/apk/debug/app-debug.apk`.

---

## Permisos y primera configuración

Al abrir por primera vez, concede:

| Permiso | Para qué |
|---|---|
| **Ubicación (precisa)** y **en segundo plano** | GPS con pantalla apagada |
| **Teléfono** (`READ_PHONE_STATE`) | Leer info de celda (CID, PCI, etc.) |
| **Notificaciones** | Notificación del servicio de grabación |

La **primera vez que grabas**, la app ofrece **excluirla de la optimización de batería**.
Acéptalo: es necesario para grabaciones largas sin que el sistema la pause.

---

## Las ventanas (pestañas)

La app tiene **4 pestañas** en la barra superior: **Mapa · Red · iPerf3 · Gráfica**.

### 1. Mapa
Rastro de la ruta sobre **OpenStreetMap**, con cada punto coloreado según una métrica:

- **RSRP** — nivel de señal de la servidora.
- **SINR** — calidad.
- **DL** — throughput de bajada.

La posición se sigue en vivo mientras grabas. Sirve para ver dónde hubo buena/mala
cobertura y correlacionar con la ruta.

### 2. Red
Es la pantalla de diagnóstico de radio. Arriba, un **panel** con la celda servidora
(tecnología, RSRP, SINR, operador y PLMN). Debajo, el **detalle completo**:

- Estado del servicio, roaming, red, SIM, PLMN (MCC/MNC), GPS.
- Campos de celda: **NodeID (gNB/eNB), NCI/CID, LCID, PCI, TAC, banda/ARFCN, BW, RSRP, RSRQ, SINR, RSSI, CQI, TA**.
- En **NSA** muestra la **NR + la LTE ancla**; en **SA** muestra **solo la NR**.
- Lista de **celdas vecinas** (cuando el módem las expone) e información de **agregación de portadoras (CA)**.

### 3. iPerf3
Es donde se **construye el script** de tráfico y se lanza la medición:

- **Nombre del UE** — prefija la carpeta de la sesión y aparece en `config.txt` (identifica cada equipo).
- **Ping** — destino del ping en paralelo (vacío = IP del primer bloque iPerf3).
- **Vídeo** — ID/URL de YouTube para Vídeo QoE en paralelo (vacío = sin vídeo).
- Botones **`+UL` `+DL` `+UDP` `+Idle`** para añadir bloques, y **`ⓘ Cómo configurar`** con ayuda.
- **Consola en vivo** y resumen de la última muestra.
- Botón de **Iniciar / Detener medición**.

### 4. Gráfica
Series temporales de las métricas (**RSRP, SINR, throughput, jitter**, etc.) con el eje de
tiempo en **hora real**, útil para ver la evolución y los eventos (handovers, caídas).

---

## Configurar el script

Cada bloque corre **en secuencia y en bucle** hasta que detienes la medición.

| Botón | Tipo | Qué hace |
|---|---|---|
| `+UL` | iPerf3 TCP subida | Carga de subida; pregunta streams `-P` |
| `+DL` | iPerf3 TCP bajada | Añade `-R` (reverse); pregunta `-P` |
| `+UDP`| iPerf3 UDP subida | Reporta **jitter y pérdida** (`-u -b`) |
| `+Idle`| Reposo | Sin tráfico; solo radio/GPS/ping etiquetado "Reposo" |

### Estructura del comando iPerf3

> En el campo **parámetro** escribes **solo lo que va después de `-c`**.
> La app añade sola: `-c`, `-i 1`, `-t <duración>`, `--forceflush`, y `-R` si es DL.

| Flag | Significado |
|---|---|
| `-c IP` | Destino (servidor) |
| `-p PUERTO` | Puerto |
| `-u` | UDP (sin `-u` = TCP) |
| `-b 50M` | Tasa objetivo (cap UDP / límite TCP). **Es por stream** |
| `-P 8` | Streams paralelos (solo TCP; llena la tubería) |
| `-R` | Reverse = bajada (lo pone `+DL`) |
| `-l 1000` | Tamaño de paquete |
| `-t N` | Duración (campo "duración" del bloque) |

**Bloques por defecto:** UL y DL a `10.20.12.48 -p 5201 -P 8` (20 s) y UDP a
`10.45.0.1 -p 5201 -u -b 50M` (20 s).

> **Slice de red:** iPerf3 no tiene flag de "slice". El slice lo decide la **ruta**
> (APN/DNN o la IP/puerto del servidor que enruta por ese slice). Para apuntar a un slice
> concreto solo cambia el **destino** en el parámetro.

---

## Ejemplo con servidores iPerf3 públicos

Si no tienes servidor propio, puedes validar un equipo contra servidores públicos.
Lista mantenida: **https://github.com/R0GGER/public-iperf3-servers** (comprueba allí cuál
está activo). Algunos (prioriza el más cercano a ti):

| Servidor | Puerto | Ubicación |
|---|---|---|
| `185.93.3.50` | 5201 | Madrid, España |
| `iperf.online.net` | 5200–5209 | París, Francia |
| `speedtest.wobcom.de` | 5201–5210 | Berlín, Alemania |
| `speedtest.ams1.nl.leaseweb.net` | 5201–5210 | Ámsterdam, Países Bajos |

### Configurarlo en la app (paso a paso)

**Bajada TCP contra Madrid:**
1. Pestaña **iPerf3** → botón **`+DL`**.
2. En **parámetro** escribe:
   ```
   185.93.3.50 -p 5201 -P 4
   ```
3. Duración, p.ej. `20`. Pulsa **Añadir**.
4. Pulsa **Iniciar medición**.

**Subida TCP contra París:**
- Botón **`+UL`**, parámetro:
  ```
  iperf.online.net -p 5201 -P 4
  ```

**UDP (jitter/pérdida) contra París:**
- Botón **`+UDP`**, parámetro:
  ```
  iperf.online.net -p 5201 -u -b 20M
  ```

### Equivalente por ADB

```bash
adb -s <serie> shell am start -n com.netmon.osm/.ui.MainActivity \
  --es blocks "IPERF_DL|185.93.3.50 -p 5201 -P 4|20; IPERF_UL|iperf.online.net -p 5201 -P 4|20" \
  --ez rec true
```

> ⚠️ **Los servidores públicos son compartidos, lejanos y con límites** (duración corta,
> pocos `-P`, a veces capados, y **muchos bloquean UDP**). Sirven para una **prueba
> funcional**, no para medir throughput representativo. Para medidas serias usa **tu propio
> servidor** (cercano, sin límites y con UDP abierto).

---

## Grabación

- El botón de **grabar** (flotante, o "Iniciar medición") arranca a la vez la **captura
  pasiva** (radio+GPS+throughput) y el **script** de bloques.
- Corre como **servicio en primer plano** con *wakelock* → sigue midiendo con la pantalla
  apagada o la app en segundo plano.
- **Durabilidad:** cada fila del CSV se vuelca (`flush`) y se hace **`fsync` cada 5 filas**
  → ante un cierre inesperado se pierden a lo sumo unos segundos.
- **Modo pasivo:** si dejas el script **vacío**, la app solo registra; el tráfico lanzado
  por otra herramienta se contabiliza igual en `drive.csv` (`DL_kbps`/`UL_kbps`, a nivel de
  dispositivo).

---

## Control por ADB (opcional)

Útil para campañas largas y para un *watchdog* que relance la app si el sistema la mata.

```bash
# Arrancar / detener grabacion
adb -s <serie> shell am start -n com.netmon.osm/.ui.MainActivity --ez rec true
adb -s <serie> shell am start -n com.netmon.osm/.ui.MainActivity --ez rec false

# Vaciar el script (modo pasivo)
adb -s <serie> shell am start -n com.netmon.osm/.ui.MainActivity --ez clearblocks true

# Definir el script (TIPO|param|dur, separados por ';')
adb -s <serie> shell am start -n com.netmon.osm/.ui.MainActivity \
  --es blocks "IPERF_DL|10.45.0.1 -p 5201 -P 8|20; IDLE||30"

# Verificar que bloques tiene guardados (build debug)
adb -s <serie> shell run-as com.netmon.osm cat shared_prefs/netmon_script.xml
```

Tipos válidos: `IPERF_UL`, `IPERF_DL`, `HTTP_DL`, `IDLE`, `VIDEO_QOE`.
UDP = `IPERF_UL` con `-u` en el parámetro. El script **solo se modifica si NO está grabando**.

---

## Archivos de salida (reportes)

Cada sesión crea una carpeta:

```
/sdcard/Android/data/com.netmon.osm/files/Documents/<UE>_<DDMMAAAA_HHMMSS>/
```

| Archivo | Contenido |
|---|---|
| `config.txt` | Equipo, servidor, puerto, vídeo, ping y el script usado |
| `drive.csv` | Muestreo 1/s de radio + GPS + throughput (dispositivo) + vecinas |
| `script.csv` | Muestreo 1/s ligado al bloque activo (iPerf3/ping/vídeo) |
| `report.html` | Informe HTML de la sesión (al detener limpiamente) |
| `track.kml` | Ruta en KML para Google Earth (al detener limpiamente) |

**Columnas de `drive.csv`:**
`Time, Lat, Lon, Speed_kmh, Type, State, Roaming, PLMN, NodeID, CID, LCID, PCI, TAC, ARFCN, Band, BW_MHz, RSRP, RSRQ, SINR, RSSI, CQI, TA, Level_dBm, Quality, DL_kbps, UL_kbps`
+ vecinas `N1..N6` cada una con `Ni_Tech, Ni_PCI, Ni_RSRP, Ni_RSRQ, Ni_SINR, Ni_ARFCN, Ni_Level_dBm`.

**Columnas de `script.csv`:**
`Time, Block, Lat, Lon, Tech, PCI, RSRP, RSRQ, SINR, RSSI, CQI, TH_DL_kbps, TH_UL_kbps, Video_DL_kbps, Video_UL_kbps, Ping_ms, Ping_jitter_ms, Iperf_Mbps, Jitter_ms, Loss_pct, HTTP_kbps, Video_startup_ms, Video_stalls, Video_res`.

> **Nota:** `DL_kbps`/`UL_kbps` (en `drive.csv`) miden **todo el dispositivo** → captan el
> tráfico externo. `TH_DL_kbps`/`TH_UL_kbps` (en `script.csv`) miden **solo la app**.

---

## Menú (tres puntos)

- **Optimización de batería** — comprueba o solicita la exención.
- **Ver informe de la sesión** — abre `report.html`.
- **Compartir sesión (ZIP)** — empaqueta la carpeta de la sesión.
- **¿Dónde se guardan los archivos?** — muestra la ruta y sesiones recientes.
- **Exportar KML** — comparte la ruta.
- **Ajustes** — tema claro/oscuro.
- **Información** — autoría y correo de contacto.

---

## Limitaciones conocidas

- **Vecinas NR en SA:** Android **no expone** las vecinas 5G en modo SA (solo la servidora).
  En LTE y NSA sí se capturan. No es un fallo de la app sino del API/módem.
- **RNTI:** no accesible por el API público de Android (vive en la capa MAC del módem).
- **Grabaciones muy largas:** algunos fabricantes (p.ej. Samsung) pueden matar procesos en
  segundo plano pese a las exenciones. Para campañas de muchas horas se recomienda un
  *watchdog* por ADB que relance la app si cae.

---

## Contacto

Aplicación desarrollada por **Ricardo Barrios** — [rnbarmuo@iteam.upv.es](mailto:rnbarmuo@iteam.upv.es)
Instituto de Telecomunicaciones y Aplicaciones Multimedia (iTEAM) — Universitat Politècnica de València.
