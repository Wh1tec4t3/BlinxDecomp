# Blinx: The Time Sweeper — Decomp
Documentación del proyecto de ingeniería inversa y port a PC.

## Herramientas
- Ghidra 10.1
- xemu
- extract-xiso

////////////////////////////////////////////////////

## Sistema de archivos mfCi
Clase de Artoon para manejo de archivos con sistema de handles propio.

### Métodos identificados
- mfCiOpen   → 00102480 — abre archivo, valida nombre y modo
- mfCiClose  → 00102860 — cierra y limpia handle
- mfCiSeek   → 00102520 — mueve posición (SEEK_SET/CUR/END)
- mfCiTell   → 001025b0 — retorna posición actual
- mfCiReqRd  → 001025e0 — lee sectores a buffer

### Estructura mfCiHandle (incompleta, WIP)
offset 0x00 → is_open (char)
offset 0x01 → status (char) — 0=libre, 1=listo, 2=leyendo
offset 0x04 → sector_size (int)
offset 0x08 → desconocido
offset 0x0c → file_size (int)
offset 0x10 → position (int)
offset 0x14 → bytes_read (int)
offset 0x18 → sectors_read (int)
offset 0x1c → filename (char[])
offset 0x30 → read_dest (int)
offset 0x34 → read_size (int)

### Vtable en 0037d778
0037d778 → FUN_000b4750  (desconocido)
0037d77c → FUN_001023c0  (desconocido)
0037d780 → FUN_001023e0  (desconocido)
0037d788 → mfCiOpen      ✓
0037d78c → mfCiClose     ✓
0037d790 → mfCiSeek      ✓
0037d794 → mfCiTell      ✓
0037d798 → mfCiReqRd     ✓

////////////////////////////////////////////////////

### Vtable completa (0037d778)
0037d778 → mfCiFile_destructor   — destructor vacío
0037d77c → mfCiSetErrorCallback  — configura sistema de logging
0037d780 → mfCiGetSector         — parsea nombre de archivo (formato XXXXXXXX.YYYYYYYY)
0037d788 → mfCiOpen              ✓
0037d78c → mfCiClose             ✓
0037d790 → mfCiSeek              ✓
0037d794 → mfCiTell              ✓
0037d798 → mfCiReqRd             ✓

### Formato de nombres de archivo
- Exactamente 17 caracteres: XXXXXXXX.YYYYYYYY
- Ambas partes son números hexadecimales
- Posición 8 siempre es un punto '.'

////////////////////////////////////////////////////

## Sistema de Tiempo (world_update - 000a6970)

### Variables principales
- world_time    (DAT_0050c81c) — tiempo acumulado del mundo, máx 17999.0
- time_speed    (DAT_0050c818) — velocidad del tiempo
- time_scale    (DAT_0050a9e4) — escala de tiempo para física
- real_time     (DAT_0050a9a4) — contador de frames reales, máx 19800
- time_power    (DAT_0050a988) — estado del poder de tiempo activo
- is_replay     (DAT_0050aae4) — 0=grabar, 1=reproducir

### Velocidades de tiempo
 1.0  = normal
 0.25 = Slow
-1.0  = Reverse
-2.0  = Reverse rápido  
 2.0  = Forward
 0.0  = Pause

### Sistema de replay
- Graba estado del jugador cada frame en replay_buffer (DAT_0050d458)
- Buffer indexado por real_time * 0x3c
- Reproduce exactamente en modo replay

### Límites
- world_time máximo: 17999.0
- real_time máximo: 19800 frames (660 segundos a 30fps)

////////////////////////////////////////////////////

## Tabla de estados del juego (001d0bc0)
Cada entrada ocupa 0x20 bytes

01 → StageStart      — inicio del nivel
02 → StageClear      — nivel completado  
03 → TimeOver        — tiempo agotado
04 → GateOpen        — puerta abierta
05 → tomSTART        — aparicion de los Tom Tom Gang
06 → tomEND          — fin de aparicion de los Tom Tom Gang
07 → Get             — recoger item
08 → NotGet          — item no obtenido
09 → TimeStop        — poder Pause
10 → TimeFF          — poder Fast Forward
11 → TimeSlow        — poder Slow
12 → TimeRwd         — poder Rewind
13 → TimeRec         — poder Record
14 → Retry           — reintentar
15 → TimeControlEnd  — fin control de tiempo
16 → good            — resultado bueno
17 → bad             — resultado malo

////////////////////////////////////////////////////

## Sistema de niveles (load_level - 000bd570)

### IDs de niveles
0x00-0x03 → Mundos de cielo (sora)
0x04-0x07 → Mundos de madera/plata
0x08-0x0b → Mundos de hongos (kinoko)
0x0c-0x0f → Mundos de castillo (taimatu)
0x10-0x13 → Mundos de acantilado (gake)
0x14-0x16 → Mundos industriales (tetu)
0x17      → Mundo de rocas (iwa)
0x1c-0x1f → Mundos de hielo (ice/aurola)
0x20-0x23 → Mundos especiales
0x24      → Nivel especial (pilares)
0x25      → Nivel especial (escalones)
0x27      → Último jefe (last boss)

### Reutilización de assets
0x18 → usa assets de 0x13
0x19 → usa assets de 0x1f
0x1a → usa assets de 0x17
0x1b → usa assets de 0x23

### Audio por mundo
Cada 4 niveles comparten audio: media\10_rd[N]se.bin
Jefes mundos 1/3: media\8_boss13.bin
Jefes mundos 2/4: media\8_boss24.bin
Último jefe:      media\8_lasbos.bin
Enemigos normales: media\7_enese.bin