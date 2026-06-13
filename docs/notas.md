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