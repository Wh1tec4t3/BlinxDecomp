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

////////////////////////////////////////////////////

## Inicialización de nivel (init_level - 000a5cd0)

- Resetea world_time, real_time_counter, time_expired_flag a 0
- Llama a load_level(DAT_0050af48) — ID del nivel seleccionado
- DAT_0050af48 = nivel actualmente seleccionado en menú

### Nombres de mundos internos
current_level_id / 4:
  0 → round + "1d" (0x3164)
  1 → round + "2d"
  2 → round + "3d"
  3 → round + "4d"
  4 → round + "5d"
  5 → round + "6d"
  7 → round + "8d"
  8 → round + "9d"
  0x24 → "BossDemo"

### Jefes por mundo
  mundo 0   → boss_a (003784f0)
  mundos 1-2 → boss_a (00378450)
  mundos 3-4 → boss_a (003785b8)
  mundo 5   → boss_a (003786a8)
  mundos 7-8 → boss_a (00378900)
  último    → lasbos01 (00378a68)

////////////////////////////////////////////////////

## Sistema de tienda

### load_shop_assets (000e0df0)
Carga todos los assets de la tienda al entrar.

### render_shop (000e1a50)
Renderiza la UI de la tienda cada frame.

### Items identificados
- i_s_payment_xx   → moneda principal
- i_s_hearts_xx    → corazones
- i_s_retryheart_xx → corazón de retry
- i_s_TS-1000_xx   → modelo sweeper TS-1000
- i_s_timebowl_xx  → recipiente de tiempo
- i_s_lifevessel_xx → recipiente de vida

### Trajes (jackets)
z_p_jacket_[rd/am/sb/or/ri/st/cm/zb/tc/mt]
10 variantes de color del traje de Blinx

### Archivos de tienda
shop01_xx → UI principal
shop02_xx → moneda
shop03_xx → items especiales
shop05_xx → corazones
shop_com  → assets comunes

### Grid de items
5 columnas × N filas
Separación: 35px horizontal, 26px vertical

### Slots de equipo
5 slots simultáneos (DAT_004c4b80 al DAT_004c4b90)

////////////////////////////////////////////////////

## Sistema de Input (read_gamepad_input - 000f25d0)

### Función principal
- Lee estado de hasta 4 controles simultáneos
- Usa XInputGetState de Xbox → equivalente SDL_GameControllerGetAxis en PC

### Ejes del control
- analog_x/y → stick izquierdo (movimiento del jugador)
- analog_z/w → stick derecho (control de cámara)
- Deadzone: 25% (ignora valores < 0.25)
- Normalización: valor_entero * (1/32767) → float [-1.0, 1.0]

### Para el port
SDL2 reemplaza XInput directamente:
- XInputOpen     → SDL_GameControllerOpen
- XInputGetState → SDL_GameControllerGetAxis
- XInputClose    → SDL_GameControllerClose

## Sistema de Input del Jugador (process_player_input - 0009ebc0)
- Lee DAT_003e08a0 con stride 0x1e4 por jugador
- DAT_003eab90 = índice del jugador activo
- Combina input de hasta 4 controles (OR de botones)
- Clamp de ejes a [-32768, 32767]
- Escribe resultado procesado de vuelta a DAT_003e08a0

////////////////////////////////////////////////////

## Sistema de entidades del jugador

### Vtable del jugador (encontrada en runtime en 003e5b31)
offset 0x6c → FUN_000b4690  — dispatcher de comportamiento
offset 0x70 → player_entity_update (000b46e0)
offset 0x74 → LAB_000b4c80
offset 0x78 → mfCiFile_destructor

### Constructor del jugador (000b4d40)
- Inicializa vtable y valores base
- offset 0x08 → escala inicial 1.0f
- offset 0x54-0x5c → velocidad XYZ inicial 0
- offset 0x60/0x61 → flags de estado activo

### Tabla de estados de comportamiento (001ddfa0)
estado 0 → NULL
estado 1 → 000b4a30 — sistema de succión del sweeper
estado 2 → 000b47c0 — animaciones del jugador
estado 3 → 000b49a0 — stub vacío
estado 4 → NULL
estado 5 → 000b4760 — transición flag=1
estado 6 → 000b4780 — transición flag=2 + sonido (ID 0x200d)
estado 7 → 000b4990 — transición flag=3
estado 8 → 000b4d40 — constructor
estado 9 → 000b4c80 — desconocido

### Nota
Las físicas reales de movimiento/gravedad no están en esta tabla.
Probablemente en FUN_000aeb10 — llamada 8 veces en FUN_000b4690
con datos desde offset 0x3e de la entidad.