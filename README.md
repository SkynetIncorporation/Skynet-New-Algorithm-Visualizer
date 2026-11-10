# Huffman Web Compressor — Skynet

Herramienta web de compresión de texto basada en el **algoritmo de Huffman**, una técnica de optimización voraz. Implementa un formato de archivo propio (`.skyh`), visualización paso a paso del árbol de Huffman, inspección de archivos comprimidos e integración con Project Gutenberg.

**Actividad Integradora | Análisis de Algoritmos | 2026B**

---

## 1. Funcionalidades principales

- **Entrada de texto:** permite escribir texto directamente, cargar archivos `.txt` o buscar libros en Project Gutenberg mediante título, autor o palabras clave.
- **Algoritmo de Huffman:** incluye cálculo de frecuencias, cola de prioridad (min-heap), construcción del árbol, generación de códigos, codificación y decodificación.
- **Formato propio:** genera archivos comprimidos con extensión `.skyh`, sin utilizar bibliotecas externas de compresión como ZIP o GZIP.
- **Visualización:** presenta las frecuencias de los símbolos, la construcción animada del árbol y los códigos asignados.
- **Métricas:** muestra tamaño original, tamaño comprimido, ahorro, porcentaje de reducción, razón de compresión y cantidad de símbolos diferentes.
- **Inspector de archivos:** permite consultar la estructura de archivos `.skyh` y realizar su descompresión.
- **Integridad:** utiliza SHA-256 para comprobar que el contenido recuperado sea idéntico al original.

### Flujo de funcionamiento

```text
BUSCAR LIBRO / SUBIR TXT
          |
          v
       ANALIZAR
          |
          v
  CONSTRUIR HUFFMAN
          |
          v
      COMPRIMIR
          |
          v
     INSPECCIONAR
          |
          v
DESCARGAR / DESCOMPRIMIR
          |
          v
VERIFICAR INTEGRIDAD
```

---

## 2. Descripción del algoritmo de Huffman

El algoritmo de Huffman asigna códigos binarios de longitud variable a los símbolos de un conjunto, utilizando códigos más cortos para los símbolos que aparecen con mayor frecuencia.

La implementación sigue estos pasos:

1. **Cálculo de frecuencias:** se recorre el contenido de entrada y se cuenta cuántas veces aparece cada byte, considerando valores de 0 a 255.

2. **Construcción del árbol:** cada símbolo se representa mediante un nodo hoja dentro de una cola de prioridad (min-heap). Mientras exista más de un nodo, se extraen los dos de menor frecuencia y se combinan en un nuevo nodo padre cuya frecuencia corresponde a la suma de ambos.

3. **Asignación de ramas:** el nodo con menor frecuencia se coloca en la rama izquierda (`0`) y el siguiente en la derecha (`1`).

4. **Desempate determinista:** cuando dos nodos tienen la misma frecuencia, se utiliza el símbolo mínimo de sus respectivos subárboles para establecer un orden consistente. Esto permite reconstruir el mismo árbol a partir de la tabla de frecuencias.

5. **Generación de códigos:** se recorre el árbol acumulando los valores `0` y `1` de cada rama. Los símbolos frecuentes reciben códigos más cortos y los menos frecuentes reciben códigos más largos. Cuando existe un único símbolo diferente, se utiliza el código `0`.

6. **Codificación:** cada byte se sustituye por su código binario. Posteriormente, los bits se empaquetan directamente en bytes, colocando el primer bit en la posición más significativa.

7. **Decodificación:** se reconstruye el árbol utilizando la tabla de frecuencias y se recorren los bits del archivo comprimido hasta recuperar el tamaño original del contenido.

### Decisión de diseño

La implementación utiliza el **byte como unidad de procesamiento**, en lugar del carácter Unicode. Esto permite recuperar exactamente el contenido original, incluyendo acentos, caracteres como la ñ, BOM y saltos de línea `\r\n`.

### Complejidad algorítmica

Considerando:

- `n`: cantidad de bytes del contenido original.
- `k`: cantidad de símbolos diferentes, donde `k <= 256`.

| Operación | Complejidad |
|---|---|
| Cálculo de frecuencias | O(n) |
| Construcción del árbol | O(k log k) |
| Codificación | O(n) |
| Decodificación | O(n) |

---

## 3. Formato de archivo propio: `.skyh`

El proyecto implementa un formato binario propio denominado **SKYH**, actualmente en su versión 1. La especificación completa está en [`docs/FORMATO.md`](docs/FORMATO.md).

La estructura general del archivo es:

```text
[ HEADER 48 B ][ NOMBRE ][ TABLA DE FRECUENCIAS ][ DATOS COMPRIMIDOS ]
```

### Estructura del encabezado

| Offset | Bytes | Campo |
|---|---:|---|
| 0 | 4 | Identificador Magic: `SKYH` (`53 4B 59 48`) |
| 4 | 1 | Versión del formato (`1`) |
| 5 | 1 | Flags reservados (`0`) |
| 6 | 4 | Tamaño original en bytes (`uint32 LE`) |
| 10 | 4 | Total de bits válidos de los datos (`uint32 LE`) |
| 14 | 2 | Cantidad de símbolos diferentes (`uint16 LE`, 0–256) |
| 16 | 32 | Hash SHA-256 del contenido original |
| 48 | 1 + L | Longitud del nombre y nombre original en UTF-8 (máximo 255 bytes) |
| Variable | N x 5 | Tabla de frecuencias: 1 byte de símbolo y 4 bytes de frecuencia (`uint32 LE`) |
| Variable | Variable | Datos comprimidos, con bits empaquetados en orden MSB primero |

### Decisiones de diseño

- **Tabla de frecuencias:** se almacena en lugar del árbol serializado. Gracias al desempate determinista, el árbol puede reconstruirse de manera idéntica.
- **Último byte:** el campo de bits válidos indica cuántos bits deben procesarse. El relleno se calcula mediante `(8 - bits mod 8) mod 8` y se completa con ceros.
- **Integridad:** el hash SHA-256 del contenido original se almacena en el encabezado y se vuelve a calcular durante la descompresión.
- **Contenido original:** el texto sin comprimir no se almacena dentro del archivo `.skyh`.

### Validaciones del formato

Al abrir un archivo `.skyh`, se realizan las siguientes comprobaciones:

- Identificador Magic correcto.
- Versión del formato compatible.
- Tamaño total del archivo esperado.
- Tabla de frecuencias ordenada y sin frecuencias iguales a cero.
- Suma de frecuencias igual al tamaño original.
- Cantidad de bits coherente con los códigos derivados de la tabla.

### Casos especiales

- **Un solo símbolo diferente:** se utiliza el código `0`.
- **Texto vacío:** se establece `N = 0` y no se almacenan datos comprimidos.

---

## 4. Instrucciones de ejecución

### Ejecución local

El proyecto no requiere compilación ni instalación de dependencias externas para ejecutar la aplicación.

Clonar el repositorio:

```bash
git clone https://github.com/SkynetIncorporation/Skynet-New-Algorithm-Visualizer.git
cd Skynet-New-Algorithm-Visualizer
```

Existen dos alternativas para ejecutar la aplicación:

**Opción 1.** Abrir directamente el archivo `huffman_visualizer.html` en un navegador.

**Opción 2.** Iniciar un servidor local, opción recomendada para evitar restricciones relacionadas con WebCrypto y CORS al utilizar `file://`.

```bash
python -m http.server 8000
```

Posteriormente, acceder a:

`http://localhost:8000/huffman_visualizer.html`

**Importante:** el archivo `skyh.js` debe encontrarse en la misma carpeta que el archivo HTML.

### Ejecución de pruebas

Para ejecutar las pruebas automatizadas:

```bash
node --test
```

Se requiere Node.js 18 o superior.

---

## 5. Tecnologías utilizadas

| Tecnología | Aplicación |
|---|---|
| JavaScript (ES2020) | Implementación del algoritmo de Huffman y lógica de compresión |
| HTML5 | Estructura de la interfaz |
| CSS3 | Diseño y estilos de la aplicación |
| SVG | Representación gráfica del árbol de Huffman |
| WebCrypto | Generación de hashes SHA-256 |
| JavaScript puro | Respaldo para la generación de SHA-256 |
| Gutendex API | Búsqueda de libros y textos |
| Project Gutenberg | Fuente de los textos utilizados |
| Cloudflare Workers | Proxy CORS |
| GitHub Pages | Despliegue de la aplicación |
| Node.js | Ejecución de pruebas automatizadas mediante `node --test` |

---

## 6. Pruebas y validación

El funcionamiento de la aplicación contempla diferentes escenarios de prueba:

| Prueba | Objetivo |
|---|---|
| Texto pequeño | Verificar manualmente frecuencias y códigos generados |
| Libro de Project Gutenberg | Comprobar el funcionamiento con archivos reales de mayor tamaño |
| Integridad | Verificar que el archivo descomprimido sea idéntico al original mediante SHA-256 y comparación byte a byte |
| Archivo nuevo | Comprobar el funcionamiento con entradas diferentes a las utilizadas durante el desarrollo |
| Casos especiales | Probar un solo símbolo, texto vacío, UTF-8 con BOM, saltos `\r\n` y bytes aleatorios |
| Archivo corrupto | Comprobar que el inspector rechace archivos con identificadores, versiones o tamaños inválidos |

Las evidencias del proyecto, incluyendo capturas de pantalla, salidas de consola y archivos de ejemplo, se encuentran en la carpeta `evidencias/`.

---

## 7. Uso de inteligencia artificial

Durante el desarrollo del proyecto se utilizaron herramientas de inteligencia artificial como apoyo en actividades de programación, revisión de código, diseño y documentación.

El código generado o modificado con asistencia de inteligencia artificial fue revisado y sometido a pruebas para comprobar su funcionamiento. Las decisiones de implementación y los resultados pueden justificarse técnicamente.

---

## 8. Limitaciones conocidas

- **Sobrecarga en archivos pequeños:** el encabezado tiene un costo fijo de 48 bytes, además de la tabla de frecuencias, que puede ocupar hasta 1,280 bytes.
- **Tamaño máximo:** la versión 1 admite archivos de hasta 4 GB debido al uso de campos `uint32`.
- **Compresión por frecuencia:** Huffman por byte no aprovecha patrones repetidos entre palabras, por lo que no ofrece las mismas capacidades que algoritmos como LZ77 o compresores por bloques.
- **Restricciones de CORS:** Project Gutenberg no proporciona las cabeceras CORS necesarias para todas las solicitudes desde el navegador. La descarga depende de un proxy externo que puede presentar restricciones o interrupciones.
- **Visualización de archivos grandes:** cuando se procesan libros de gran tamaño, la animación del árbol y de la codificación se limita a una parte del contenido para mantener un rendimiento adecuado. Sin embargo, las frecuencias y el archivo `.skyh` utilizan el contenido completo.

---

## 9. Estructura del repositorio

```text
.
├── huffman_visualizer.html   # Interfaz y visualización paso a paso
├── skyh.js                   # Núcleo Huffman, formato .skyh y SHA-256
├── test/                     # Pruebas automatizadas
├── evidencias/               # Capturas y resultados de pruebas
├── docs/
│   └── FORMATO.md            # Especificación del formato .skyh
└── README.md                 # Documentación principal
```

---

## 10. Gestión del proyecto

La organización y seguimiento de las actividades se realiza mediante **GitHub Projects**.

El tablero utiliza las siguientes columnas para representar el estado de las tareas:

```text
BACKLOG → POR HACER → EN PROGRESO → EN REVISIÓN → TERMINADO
```

Cada tarea contiene una descripción del trabajo por realizar, el resultado esperado y su estado de avance.

---

## 11. Licencia y créditos

Este proyecto fue desarrollado con fines académicos como parte de la Actividad Integradora de la asignatura Análisis de Algoritmos, correspondiente al periodo 2026B.

Los textos utilizados como fuente de prueba provienen de [Project Gutenberg](https://www.gutenberg.org/), mientras que las búsquedas de libros se realizan mediante la API de [Gutendex](https://gutendex.com/).
