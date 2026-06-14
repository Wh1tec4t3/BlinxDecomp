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

////////////////////////////////////////////////////

## Sistema de entidades del jugador

### Vtable del jugador (encontrada en runtime en 003e5b31)
offset 0x6c → player_movement_dispatch (000b4690) — dispatcher de comportamiento
offset 0x70 → player_entity_update (000b46e0)
offset 0x74 → LAB_000b4c80
offset 0x78 → mfCiFile_destructor

### Constructor del jugador (000b4d40)
- Inicializa vtable y valores base
- offset 0x08 → escala inicial 1.0f
- offset 0x54-0x5c → velocidad XYZ inicial 0
- offset 0x60 → movement_state (uint8) — estado actual de movimiento
- offset 0x61 → movement_state_prev (uint8) — estado anterior
- offset 0x62 → movement_substate (ushort) — sub-estado interno

### Struct de entidad Blinx (base: blinx_entity_array, stride 0x1e4)
offset 0x00 → object_type (short) — 0x6e = jugador
offset 0x02 → buttons[8] (byte[8])
offset 0x0a → stick_x (short)
offset 0x0c → anim_counter (float)
offset 0x0e → stick_y (short)
offset 0x60 → movement_state (uint8)
offset 0x61 → movement_state_prev (uint8)
offset 0x62 → movement_substate (ushort)
offset 0x7c → active_nodes[8] (void*[8]) — punteros a nodos activos

### Tabla de estados de comportamiento (001ddfa0)
estado 0 → NULL
estado 1 → 000b4a30 — pit kill handler (zona específica, coords hardcodeadas)
                       substate en +0x62, kill via FUN_000dead0
estado 2 → 000b47c0 — animaciones del jugador
estado 3 → 000b49a0 — stub vacío
estado 4 → NULL
estado 5 → 000b4760 — player_state_reset (escribe movement_state=1, limpia effect_spawned_flag)
estado 6 → 000b4780 — transición flag=2 + sonido (ID 0x200d)
estado 7 → 000b4990 — transición flag=3
estado 8 → 000b4d40 — constructor
estado 9 → 000b4c80 — desconocido

### Nota
Las físicas reales de movimiento/gravedad no están en esta tabla.
FUN_000aeb10 es un entity_node_iterator — procesa 8 punteros a nodos
desde offset +0x7c de la entidad. Físicas probablemente en el caller
de player_movement_dispatch (subir en el call tree).

////////////////////////////////////////////////////

## Sistema de Input del Jugador (process_player_input - 0009ebc0)
- Lee blinx_entity_array (003e08a0) con stride 0x1e4 por entidad
- active_entity_index (003eab90) — índice de entidad activa (0-3)
- replay_mode (003e0aa4) — 0 = entidades grabadas activas, 1 = control normal
- En replay_mode=0: acumula input de hasta 4 entidades simultáneas (OR de botones)
- En replay_mode=1: usa active_entity_index directamente
- Clamp de ejes a [-32768, 32767]
- Clamp de ejes flotantes a [-1.0, 1.0]
- Escribe resultado procesado de vuelta a blinx_entity_array
- FUN_000f2310 = button_edge_detector (procesa pressed/released por frame)
- blinx_entity_array soporta hasta 4 entidades simultáneas (poderes de tiempo)

### Globals de input procesado
003e08ca → input_stick_x (short)
003e08cc → input_stick_y (short)
003e08ce → input_stick_rx (short)
003e08d0 → input_stick_ry (short)
003e08d4 → input_axis_lx (float, clamp ±1.0)
003e08d8 → input_axis_ly (float, clamp ±1.0)
003e08dc → input_axis_rx (float, clamp ±1.0)
003e08e0 → input_axis_ry (float, clamp ±1.0)
003e08c0 → input_buttons (ushort)

////////////////////////////////////////////////////

## Sistema de entidades del jugador

### Call tree completo
game_main
  → gameplay_update
    → physics_and_entity_update (000a9120)
      → physics_set_gravity (0002b760) — vector (0, -1, 0)
        → physics_propagate_gravity (0002ec60) — copia a globals
          → physics_apply_gravity (0002b7d0) — escribe al engine
      → camera_update (00024650)
      → shader_constants_setup (0002ec00)
    → gameplay_hud_update (000a9260)
      → entity_list_update (000b3ff0) — itera DAT_006304a0
        → entity->vtable[0x74] = player_entity_update (000b46e0)
          → entity->vtable[0x6c] = player_movement_dispatch (000b4690)
            → PTR_001ddfa0[movement_state](entity)

### Vtable del jugador (embebida en entidad)
offset 0x6c → player_movement_dispatch (000b4690) — dispatcher de comportamiento
offset 0x70 → player_entity_update (000b46e0) — update de animación
offset 0x74 → LAB_000b4c80 — desconocido, llamado por entity_list_update
offset 0x78 → mfCiFile_destructor

### Constructor del jugador (000b4d40)
- Inicializa vtable y valores base
- offset 0x08 → escala inicial 1.0f (0x3f800000)
- offset 0x0c → 0 (desconocido)
- offset 0x02 → 0 (desconocido, short)
- offset 0x20 → bloque inicializado por FUN_000947a0
- offset 0x40 → bloque inicializado por FUN_000947a0
- offset 0x54-0x5c → velocidad XYZ inicial 0
- offset 0x60 → movement_state = 1 (inicial)
- offset 0x61 → movement_state_prev = 1 (inicial)
- offset 0x62 → movement_substate = 0
- offset 0x6c → vtable: player_movement_dispatch
- offset 0x70 → vtable: player_entity_update
- offset 0x74 → vtable: player_entity_update
- offset 0x78 → vtable: mfCiFile_destructor
- offset 0x7c → active_nodes[8] inicializados a 0
- offset 0x9c → 0 (desconocido)

### Struct de entidad Blinx (base: blinx_entity_array 003e08a0, stride 0x1e4)
offset 0x00 → object_type (short) — 0x6e = jugador
offset 0x02 → buttons[8] (byte[8])
offset 0x0a → stick_x (short)
offset 0x0c → anim_counter (float)
offset 0x0e → stick_y (short)
offset 0x60 → movement_state (uint8)
offset 0x61 → movement_state_prev (uint8)
offset 0x62 → movement_substate (ushort)
offset 0x63 → state_flags (byte) — bit 0x20=FLAG_FROZEN, bit 0x40=FLAG_SKIP_UPDATE
offset 0x6c → vtable_movement (code*)
offset 0x70 → vtable_update (code*)
offset 0x74 → vtable_entity_update (code*)
offset 0x78 → vtable_destructor (code*)
offset 0x7c → active_nodes[8] (void*[8])
offset 0x9c → desconocido
offset 0x1ea → alive_flag (char)
offset 0x244 → secondary_counter (int)

### Nodo de lista enlazada (DAT_006304a0)
offset 0x04 → next (node*)
offset 0x08 → entity_ptr (void*)

### Tabla de estados de comportamiento (001ddfa0)
estado 0 → NULL
estado 1 → 000b4a30 — pit kill handler (zona específica, coords hardcodeadas)
                       substate en +0x62, kill via FUN_000dead0
estado 2 → 000b47c0 — animaciones del jugador
estado 3 → 000b49a0 — stub vacío
estado 4 → NULL
estado 5 → 000b4760 — player_state_reset (escribe movement_state=1, limpia effect_spawned_flag)
estado 6 → 000b4780 — transición flag=2 + sonido (ID 0x200d) — sin verificar
estado 7 → 000b4990 — transición flag=3 — sin verificar
estado 8 → 000b4d40 — constructor
estado 9 → 000b4c80 — desconocido

### Gravedad
- Vector: (0.0, -1.0, 0.0) — hardcodeado en physics_and_entity_update
- gravity_x (008d4730), gravity_y (008d4734), gravity_z (008d4738)
- Constante de escala física: 9.587378e-05
- physics_tick (009de9d4) — acumulador, += 0x147 por frame

////////////////////////////////////////////////////

## Sistema de Input del Jugador (process_player_input - 0009ebc0)
- Lee blinx_entity_array (003e08a0) con stride 0x1e4 por entidad
- active_entity_index (003eab90) — índice de entidad activa (0-3)
- replay_mode (003e0aa4) — 0 = entidades grabadas activas, 1 = control normal
- En replay_mode=0: acumula input de hasta 4 entidades simultáneas (OR de botones)
- En replay_mode=1: usa active_entity_index directamente
- Clamp de ejes a [-32768, 32767]
- Clamp de ejes flotantes a [-1.0, 1.0]
- Escribe resultado procesado de vuelta a blinx_entity_array
- button_edge_detector (000f2310) — procesa pressed/released por frame
- blinx_entity_array soporta hasta 4 entidades simultáneas (poderes de tiempo)

### Globals de input procesado
003e08ca → input_stick_x (short)
003e08cc → input_stick_y (short)
003e08ce → input_stick_rx (short)
003e08d0 → input_stick_ry (short)
003e08d4 → input_axis_lx (float, clamp ±1.0)
003e08d8 → input_axis_ly (float, clamp ±1.0)
003e08dc → input_axis_rx (float, clamp ±1.0)
003e08e0 → input_axis_ry (float, clamp ±1.0)
003e08c0 → input_buttons (ushort)

////////////////////////////////////////////////////

## Globals identificados en esta sesión

### Sistema de físicas
008d4730 → gravity_x (float) — siempre 0.0
008d4734 → gravity_y (float) — siempre -1.0
008d4738 → gravity_z (float) — siempre 0.0
009de9d4 → physics_tick (int) — acumulador, += 0x147 por frame
0050afd4 → physics_detail_level (int) — pasado a colisiones y resolución
009dea10 → physics_paused (int) — bloquea update de cámara y físicas

### Sistema de cámara
009de9a4 → cam_pos_x (float)
009de9a8 → cam_pos_y (float)
009de9ac → cam_pos_z (float)
009de934 → cam_offset_x (float)
009de938 → cam_offset_y (float)
009de93c → cam_offset_z (float)
009de9cc → cam_ang_vel_1 (float)
009de9c8 → cam_ang_vel_2 (float)
009de9d0 → cam_ang_vel_3 (float)
00667e18 → screen_scale_y (float) — escala de proyección vertical
0039ddd0 → view_matrix (float[16]) — pasada a D3DDevice_SetTransform
0050a9d0 → cam_dist_vertical (float)
0050aab4 → cam_offset_screen (float)
006305a0 → hit_entity_id (int)

### Sistema de render/shaders
008d5e60 → light0_direction (float[3])
008d5a30 → light0_color (float[3])
008762d0 → light1_direction (float[3])
00668080 → light1_color (float[3])

### Sistema de entidades
003e0aa4 → replay_mode (int) — 0=entidades grabadas activas, 1=control normal
003eab90 → active_entity_index (int) — índice de entidad activa (0-3)

### Sistema de juego
0050a9d4 → level_time_limit (float)
0050ab2c → fade_out_flag (int)
0038b008 → transition_type (int) — siempre 2 en transiciones
0050a9cc → transition_timer (int) — 0x1e = 30 frames en transiciones
0050c7b8 → retry_level_id (int)
0050a97c → cutscene_mode (int)
0050c820 → time_frozen_flag (int) — ya teníamos time_power, confirmar si es lo mismo
0050af84 → snapshot_pending (int)
0050c828 → snapshot_time (float)
009e0280 → snapshot_ready (int)
0050aaf8 → is_pal_mode (int) — controla framerate y resolución
003eab74 → frame_table_index (int) — índice en tabla de frames PAL
001dc2a8 → pal_frame_table (int[]) — tabla de frames para PAL
004c31cc → memory_card_slots (int)
0050a988 → time_power (int) — ya documentado como global_sync_state en esta sesión, nombre corregido al original
005085e8 → effect_spawned_flag (int) — ya documentado
004ebc24 → zone_object (void*) — ya documentado, offsets +0x175/0x176/0x177/0x188

### zone_object — offsets nuevos
offset 0x175 → active_powers_count (char)
offset 0x176 → time_stones_count (char)
offset 0x177 → time_stone_ids[4] (char[4])
offset 0x188 → power_slots[N] (int[]) — punteros a poderes activos