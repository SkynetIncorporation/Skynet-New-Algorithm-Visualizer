# Especificación del formato `.skyh`

Huffman Web Compressor — Skynet · Análisis de Algoritmos · 2026B · **Versión del formato: 1**

## 1. Descripción general

El formato `.skyh` es el contenedor propio con el que la aplicación guarda un texto comprimido con el algoritmo de Huffman. Un archivo `.skyh` es autosuficiente: contiene todo lo necesario para reconstruir el árbol, decodificar los datos y verificar que el resultado es idéntico al original, sin depender del archivo `.txt`.

Estructura general del archivo, en este orden:

```text
[ HEADER fijo ][ NOMBRE ][ TABLA DE FRECUENCIAS ][ DATOS COMPRIMIDOS ]
```

Convenciones:

- Todos los enteros se almacenan en **little-endian**.
- El símbolo de Huffman es un **byte (0–255)** del contenido UTF-8 del archivo, no un carácter. Así se soportan acentos, ñ y cualquier otro carácter, y la integridad se verifica byte a byte.
- Los bits de los datos comprimidos se empaquetan comenzando por el **bit más significativo (MSB)** de cada byte.

## 2. Layout del archivo

| Offset | Bytes | Campo | Propósito |
|---|---:|---|---|
| 0 | 4 | Magic `"SKYH"` (`53 4B 59 48`) | Identificar que es un archivo `.skyh` válido |
| 4 | 1 | Versión (`1`) | Permitir evolucionar el formato |
| 5 | 1 | Flags (reservado, `0`) | Extensiones futuras |
| 6 | 4 | Tamaño original (`uint32`) | Métricas y validación al descomprimir |
| 10 | 4 | Total de bits válidos (`uint32`) | Interpretar el último byte de datos |
| 14 | 2 | N: símbolos distintos (`uint16`, 0–256) | Número de entradas de la tabla |
| 16 | 32 | Hash SHA-256 del original | Verificación de integridad |
| 48 | 1 + L | Longitud L del nombre + nombre (UTF-8) | Mostrar el nombre en el inspector y sugerir el nombre del `.txt` recuperado |
| … | N × 5 | Tabla: 1 byte símbolo + 4 bytes frecuencia (`uint32`) | Reconstruir el árbol y mostrar frecuencias |
| … | ⌈bits / 8⌉ | Datos comprimidos | Secuencia de bits empaquetada |

Tamaño del header fijo: **48 bytes** (sin contar nombre ni tabla). El tamaño total esperado del archivo es:

```text
48 + 1 + L + 5·N + ⌈total_bits / 8⌉
```

## 3. Decisiones de diseño

### 3.1 Se guardan las frecuencias, no el árbol

Con la tabla de frecuencias se reconstruye exactamente el mismo árbol, y el inspector puede mostrar frecuencias, códigos y árbol sin información adicional. El costo es de a lo sumo 256 × 5 = 1280 bytes: despreciable en un libro de Project Gutenberg, pero notorio en textos muy pequeños (limitación conocida).

### 3.2 Desempate determinista

El compresor y el descompresor deben construir exactamente el mismo árbol. Se usa siempre el mismo criterio en la cola de prioridad: **menor frecuencia primero** y, ante empate, **el menor símbolo mínimo contenido en el subárbol** (o, equivalentemente, el menor orden de creación). Si los criterios difieren, los códigos cambian y el archivo no se puede decodificar.

### 3.3 Bits válidos

Se guarda el total de bits codificados. El relleno del último byte es `(8 − total_bits % 8) % 8` bits en cero. El descompresor se detiene al leer `total_bits` bits o al producir `tamaño_original` bytes; ambas cifras sirven como validación cruzada.

### 3.4 Casos borde

- **Archivo vacío:** `N = 0`, tamaño original = 0, total de bits = 0 y sin datos.
- **Un solo símbolo distinto** (por ejemplo `"aaaa"`): el árbol es una sola hoja; se le asigna el código `"0"` y `total de bits = tamaño original`.

## 4. Integridad

Al descomprimir se calcula el SHA-256 del resultado y se compara con el almacenado en el header. La interfaz muestra "idéntico" si coinciden y "corrupto" si no. Esto cubre la comprobación de integridad requerida (sección 8 de la actividad) sin necesitar el `.txt` original.

## 5. Validación al abrir un archivo

- El magic debe ser `"SKYH"`.
- La versión debe ser conocida (`1`).
- El tamaño real del archivo debe coincidir con el tamaño esperado calculado a partir del header (sección 2).
- N debe estar entre 0 y 256, y la suma de frecuencias debe ser igual al tamaño original.
- La tabla debe estar ordenada por símbolo, sin repetidos y sin frecuencias iguales a cero.
- El total de bits del header debe coincidir con el que implican los códigos derivados de la tabla.

Si alguna condición falla, el inspector muestra un mensaje de error claro en lugar de intentar continuar.

## 6. Información que obtiene el inspector

| Dato requerido por el inspector | Origen en el archivo `.skyh` |
|---|---|
| Nombre y extensión | Campo de nombre + nombre del archivo cargado (`.skyh`) |
| Versión / formato | Magic + versión |
| Tamaño original | Header (offset 6) |
| Tamaño comprimido | Tamaño real del archivo `.skyh` |
| Bytes / KB / MB ahorrados y % de reducción | Cálculo: `original − comprimido`; `(ahorro / original) × 100` |
| Número de símbolos | Header: N |
| Tabla de frecuencias | Tabla de frecuencias |
| Códigos Huffman | Reconstruidos desde la tabla |
| Árbol de Huffman | Reconstruido desde la tabla |
| Total de bits codificados | Header (offset 10) |
| Vista parcial de datos binarios | Primeros bytes de la sección de datos (hex y/o bits) |
| Descomprimir, ver y descargar `.txt` | Decodificación explícita + verificación SHA-256 |

## 7. Flujo de compresión y descompresión

### Compresión

1. Leer el texto como bytes y calcular las frecuencias.
2. Construir el árbol con una cola de prioridad (estrategia voraz) y generar los códigos.
3. Codificar los bytes y empaquetar los bits en un `Uint8Array`.
4. Calcular el SHA-256 y escribir header, nombre, tabla y datos.

### Descompresión

1. Validar el archivo (sección 5) y leer el header.
2. Leer la tabla y reconstruir el árbol con el mismo desempate.
3. Decodificar `total_bits` bits recorriendo el árbol hasta producir `tamaño_original` bytes.
4. Verificar el SHA-256 y ofrecer el texto recuperado como `.txt`.
