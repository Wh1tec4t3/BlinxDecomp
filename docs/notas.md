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
- i_s_payment_xx    → moneda principal
- i_s_hearts_xx     → corazones
- i_s_retryheart_xx → corazón de retry
- i_s_TS-1000_xx    → modelo sweeper TS-1000
- i_s_timebowl_xx   → recipiente de tiempo
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
offset 0x6c → Player_BehaviorDispatcher (000b4690) — dispatcher de comportamiento
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
Player_BehaviorDispatcher llama Node_TickSpatialSlot 8 veces por frame,
una por cada slot de emisor de partículas asociado al jugador
(efectos visuales del sweeper, partículas de tiempo, etc.)
offset 0x3e de la entidad → pendiente de analizar

////////////////////////////////////////////////////

## Sistema de partículas

Sistema completo de emisores de partículas usado por entidades del juego.
Los 8 slots del jugador controlan efectos visuales (sweeper, poderes de tiempo, etc.)

### Globals
- g_particle_emitter_table (DAT_004f13c0) — tabla de emisores, entradas de 0xc bytes
- g_emitter_flags          (DAT_004f2bc0) — flags paralelos, 1 byte por slot
- g_emitter_count          (DAT_004f2dc4) — total de slots ocupados
- g_active_emitter_count   (DAT_004f2dc0) — total de emisores activos
- g_active_emitter_table   (DAT_004ef042) — tabla de emisores activos, stride 0xc bytes

### Funciones identificadas
- Node_TickSpatialSlot      (000aeb10) — tick de un slot: valida owner, actualiza o limpia
- Node_OwnershipCheck       (dirección pendiente) — comprueba owner_id en offset 0x10 del nodo
- Node_UpdateSpatialChildren (dirección pendiente) — tick completo del nodo y sus hijos
- Node_Destroy              (000d2da0) — destructor en cadena por next_sibling
- Node_FreeSlots            (000d14e0) — libera slots en g_particle_emitter_table con compactación
- ParticleEmitter_Update    (000cf000 aprox) — actualiza partículas vivas, 3 modos de colisión
- ParticleEmitter_Spawn     (000d1720) — emite partículas nuevas desde un emisor
- ParticleEmitter_SetExpireCallback (000d0880) — configura expiración con callback
- ParticlePool_Alloc        (000cf160) — busca partícula libre en el pool
- memcpy_manual             (000de200) — copia byte a byte, equivalente a memcpy
- Timer_GetRandom           (000a2ef0) — retorna valor aleatorio o timer global

### Struct SpatialNode
offset 0x00 → flags (byte) — bit0=activo, bit1=expirando, bit2=callback, bit7=marcado sistema
offset 0x01 → child_count (byte)
offset 0x02 → slot_index (short) — índice en g_particle_emitter_table
offset 0x04 → timer (short) — countdown de vida
offset 0x06 → padding (2 bytes)
offset 0x08 → system_ref (uint*) — puntero al sistema dueño, bits 0-3 se limpian al expirar
offset 0x0A → (short) — desconocido, se limpia a 0 en Node_Destroy
offset 0x0C → padding (2 bytes)
offset 0x10 → owner_id (int) — ID del dueño, comparado en Node_OwnershipCheck
offset 0x12 → (short) — desconocido, se limpia a 0 en Node_Destroy
offset 0x14 → next_sibling (int) — siguiente nodo en lista enlazada

### Struct ParticleEmitter
offset 0x00 → flags (uint) — bits 0-3: estado, bit1=expirando, bit2=callback, bit8=modo cono, bit9=modo esfera, bits 4-7: modo de emisión (0x00/0x10/0x20/0x30)
offset 0x04 → expire_counter (short) — countdown de expiración
offset 0x06 → callback_arg1 (short)
offset 0x08 → cooldown_base (short)
offset 0x0A → eval_pending (short) — contador de partículas vivas
offset 0x0C → recalc_base (short)
offset 0x0E → cooldown_factor (ushort)
offset 0x12 → cooldown_variance (short)
offset 0x14 → cooldown_current (short)
offset 0x16 → color_base (char)
offset 0x17 → color_variance (char)
offset 0x26 → speed_min (short)
offset 0x28 → speed_max (short)
offset 0x2A → rotation_y (ushort)
offset 0x2E → rotation_z (ushort)
offset 0x32 → angle_min (ushort)
offset 0x36 → angle_min2 (ushort)
offset 0x3C → gravity_base (float)
offset 0x3E → gravity_variance (float)
offset 0x4C → spatial_ref (ParticleSpatialRef*) — referencia espacial para colisiones
offset 0x80 → block_array (Particle**) — array de punteros a bloques de partículas
offset 0x84 → block_count (uint)
offset 0x88 → slot_ptr (uint*) — puntero directo a entrada en g_particle_emitter_table
offset 0x8C → particles_per_block (uint)

### Modos de emisión (bits 4-7 de flags)
0x00 → emisión recta sin rotación
0x10 → emisión con rotación Y aleatoria
0x20 → emisión con rotación Y+Z aleatoria
0x30 → emisión con rotación Y + velocidad lateral

### Modos de colisión (bits 8-9 de flags, en ParticleEmitter_Update)
bit8=0, bit9=0 → sin colisión — movimiento libre
bit9=1         → colisión esférica — rebote dentro de esfera definida por spatial_ref
bit8=1         → colisión cónica — rebote dentro de cono definido por spatial_ref

### Struct ParticleSpatialRef
offset 0x00 → pos_x (float)
offset 0x04 → pos_y (float)
offset 0x08 → pos_z (float)
offset 0x0C → dir_x (float) — eje del cono / normal de esfera
offset 0x10 → dir_y (float)
offset 0x14 → dir_z (float)
offset 0x18 → radius_inner (float) — radio interno (esfera) / radio mínimo (cono)
offset 0x1C → radius_outer (float) — radio externo (esfera) / radio máximo (cono)
offset 0x20 → length (float) — longitud del cono
offset 0x24 → sphere_active (float) — ≠0 activa modo esfera

### Struct Particle (0xc bytes)
offset 0x00 → lifetime (char) — 0=libre, >0=viva, se decrementa cada frame
offset 0x01 → vel_x (char)
offset 0x02 → vel_y (char)
offset 0x03 → vel_z (char)
offset 0x04 → pos_x (short)
offset 0x06 → pos_y (short)
offset 0x08 → pos_z (short)
offset 0x0A → random_seed (short)

### Ciclo de vida de un nodo
ACTIVO (bit0=1)
  → decrementa timer si no está expirando
  → cuenta slots activos en g_particle_emitter_table
  → si slots > 0: sigue vivo, marca bit7
EXPIRANDO (bit1=1)
  → countdown hasta 0
  → al llegar a 0: limpia bits 0-3 de system_ref
  → propaga a hijos por next_sibling
MUERTO (active_slot_count==0 && timer_sum==0)
  → Node_Destroy → libera nodo y cadena de hijos
  → Node_TickSpatialSlot escribe NULL en el slot del jugador

////////////////////////////////////////////////////

## Sistema de colisiones (CollisionSurface)

### Arquitectura general
Sistema híbrido:
- Heightmap (grid)
- Mesh (triángulos)

### Router
Collision_TestSimple:
- transforma mundo → local
- test rápido de distancia
- delega en backend

### Heightmap
CollisionSurface_ProjectHeight_Heightmap
- float grid
- acceso height[x + z * width]
- bilinear interpolation
- terrain optimizado

### Mesh
CollisionSurface_ProjectHeight_Mesh
- vértices explícitos
- triángulos reales
- barycentric test
- mayor precisión

### Observaciones
- espacio local universal
- bounding box early-out
- heightmap O(1)
- mesh per-triangle

### Player_SuctionUpdate (0007c700)
Controla el sistema de succión del sweeper. Acumula suction_timer cada frame
y ejecuta fases según umbrales (30.0, 40.0, 49.0).

Fases:
- < 30.0  → tracking del target + actualiza ángulo sweep
- 30-40   → activa flag 0x400, calcula suction_speed, lanza partículas (tipo 1, pool 5)
- 40-49   → aplica rotación al sweeper hacia el target
- > 49.0  → si bit31 activo: countdown suction_countdown, al llegar a <0 termina succión

### Campos de la entidad del jugador (PlayerEntity struct — WIP)
offset 0x02  → short     — desconocido
offset 0x04  → float     — buffer/pos anterior
offset 0x08  → float     — scale
offset 0x0C  → ?         — desconocido (init 0)
offset 0x10  → uint      — entity_flags
                           bit10 (0x400) = succión activa
                           bit31 (0x80000000) = fase final succión
offset 0x14  → float     — pos_y
offset 0x18  → float     — pos_z
offset 0x1C  → float     — suction_timer (acumula hasta 30/40/49)
offset 0x20  → float     — suction_pos_x ⚠ conflicto con rotation_quat, pendiente confirmar
offset 0x24  → float     — sweep_angle
offset 0x28  → float     — suction_pos_z
offset 0x2C  → float     — desconocido (W del bloque en +0x20)
offset 0x30  → float     — rot_x
offset 0x34  → float     — rot_y
offset 0x38  → float     — rot_z
offset 0x40  → float[4]  — bloque aceleración/dirección (xyzw, init 0/0/0/1)
offset 0x54  → float     — vel_x (init 0)
offset 0x58  → float     — vel_y (init 0)
offset 0x5C  → float     — vel_z (init 0)
offset 0x60  → byte      — behavior_state (init 1)
offset 0x61  → byte      — behavior_state_prev (init 1)
offset 0x62  → short     — suction_state
offset 0x64  → float     — sweep_world_angle
offset 0x68  → float     — desconocido (init 0 al terminar succión)
offset 0x6C  → fn*       — vtable: Player_BehaviorDispatcher
offset 0x70  → fn*       — vtable: player_entity_update
offset 0x74  → fn*       — vtable: player_entity_render
offset 0x78  → fn*       — vtable: mfCiFile_destructor
offset 0x7C  → SpatialNode*[8] — particle_slots (init 0)
offset 0x9C  → uint      — active_particle_slot (init 0, módulo 8)
offset 0xC5  → byte      — fx_counter (valores 0x32, 0x64)
offset 0xD2  → char      — suction_duration_mult
offset 0xD4  → short     — suction_duration (= suction_duration_mult * 0x5a)
offset 0xD6  → short     — suction_countdown (decrementa, termina succión al llegar a <0)
offset 0xE4  → float     — desconocido