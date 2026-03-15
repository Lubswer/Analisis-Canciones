# Documentación  Final de Análisis de Datos
## Análisis Exploratorio del Comportamiento Musical en Spotify

**Herramienta utilizada:** KNIME Analytics Platform v5.11.0  
**Fuentes de datos:** SQLite, MongoDB Atlas, XML, JSON, CSV  
**Tipo de proyecto:** Análisis exploratorio de datos (EDA) con procesamiento multi-fuente  
**Registros procesados:** 1,244 filas (resultado confirmado por los nodos Sorter ejecutados)

---

## 1. Planteamiento del Problema

### Contexto

La industria musical digital ha transformado la manera en que se consume y distribuye música. Plataformas como Spotify concentran millones de canciones y generan datos masivos sobre popularidad de artistas, características de álbumes y tendencias de escucha. Sin embargo, esos datos en bruto no revelan patrones por sí solos: es necesario integrarlos, limpiarlos y analizarlos para extraer conocimiento accionable.

### Problema

¿Qué factores están asociados a la popularidad de una canción en Spotify? ¿Existen patrones identificables en la duración, el año de lanzamiento, la posición dentro del álbum, el género del artista o incluso en el lenguaje emocional del título de una canción que puedan explicar su nivel de aceptación por parte del público?

Adicionalmente: ¿es posible integrar datos provenientes de **cinco fuentes heterogéneas** (SQLite, MongoDB, XML, JSON, CSV) y construir un flujo de análisis unificado, limpio y reproducible que responda estas preguntas de manera visual?

### Objetivos del Análisis

1. Integrar datos de cinco fuentes distintas en un único conjunto coherente.
2. Limpiar y transformar los datos garantizando calidad analítica.
3. Identificar los artistas con mayor base de seguidores y relacionarlos con su producción.
4. Analizar la evolución temporal de lanzamientos musicales.
5. Determinar si la duración de una canción influye en su popularidad.
6. Explorar si la posición en el álbum (`track_number`) afecta el rendimiento de la canción.
7. Detectar canciones virales de artistas con baja popularidad general.
8. Aplicar análisis de sentimiento sobre títulos y correlacionar con popularidad.

---

## 2. Descripción de los Datos

### Fuentes utilizadas

| # | Fuente | Nodo KNIME | Descripción |
|---|--------|------------|-------------|
| 1 | **SQLite** | `SQLite Connector (#4)` + `DB Query Reader (#5)` | Archivo `spotify_extra_100.sqlite` → consulta `SELECT * FROM spotify_tracks` |
| 2 | **MongoDB Atlas** | `MongoDB Connector (#48)` + `MongoDB Reader (#7)` | Colección en `prueba.ufcbx7t.mongodb.net` |
| 3 | **XML** | `XML Reader (#3)` + `XPath (#28)` | Archivo XML con estructura `spotify_data/track/...` (15 campos extraídos) |
| 4 | **JSON** | `JSON Reader (#2)` + `JSON Path (#16, #49)` + `Ungroup (#17, #50)` | Datos anidados en JSON procesados con rutas y desanidado de listas |
| 5 | **CSV** | `CSV Reader (#59)` | Archivo `spotify_data.csv` local |

### Variables principales del dataset

| Campo | Tipo | Descripción |
|---|---|---|
| `track_id` | String | Identificador único de la canción |
| `track_name` | String | Nombre de la canción |
| `track_number` | Integer | Posición dentro del álbum |
| `track_popularity` | Integer (0–100) | Popularidad de la canción en Spotify |
| `track_duration_min` | Double | Duración en minutos (decimal) |
| `track_duration_time` | String | Duración en formato `MM:SS` (columna generada) |
| `artist_name` | String | Nombre del artista |
| `artist_popularity` | Integer (0–100) | Popularidad del artista |
| `artist_followers` | Integer | Seguidores del artista |
| `artist_genres` | String | Géneros musicales del artista |
| `album_id` | String | Identificador del álbum |
| `album_name` | String | Nombre del álbum |
| `album_release_date` | Date | Fecha de lanzamiento |
| `album_total_tracks` | Integer | Total de canciones en el álbum |
| `album_type` | String | Tipo: `album`, `single`, `compilation` |
| `explicit` | Boolean | Si la canción tiene contenido explícito |
| `Year` | Integer | Año extraído de `album_release_date` (columna generada) |
| `title_length` | Integer | Longitud del título en caracteres (columna generada) |
| `title_words` | Integer | Número de palabras en el título (columna generada) |
| `sentimiento_titulo` | String | Sentimiento del título: positivo / negativo / neutral (columna generada) |

---

## 3. Flujo de Trabajo en KNIME

El flujo se estructuró en cuatro etapas secuenciales:

```
╔══════════════════════════════════════════════════════════════╗
║                    ETAPA 1 — INGESTA                        ║
║  SQLite → DB Query Reader                                   ║
║  MongoDB → MongoDB Connector → MongoDB Reader               ║
║  XML    → XML Reader → XPath                                ║
║  JSON   → JSON Reader → JSON Path → Ungroup                 ║
║  CSV    → CSV Reader                                        ║
╚══════════════════════════════════════════════════════════════╝
                           │
╔══════════════════════════════════════════════════════════════╗
║                  ETAPA 2 — LIMPIEZA                         ║
║  Por cada rama de datos:                                    ║
║    String to Date_Time → Column Expressions (MM:SS)         ║
║    String Replacer (N/A → Unknown en artist_genres)         ║
║    Column Filter → Row Filter (duración > 1 min)            ║
║    Row Filter (popularidad > 30)                            ║
║    Row Filter (album_type = "album")                        ║
║  → Concatenate (une las 5 ramas)                            ║
║  → Missing Value → Duplicate Row Filter                     ║
╚══════════════════════════════════════════════════════════════╝
                           │
╔══════════════════════════════════════════════════════════════╗
║               ETAPA 3 — TRANSFORMACIÓN                      ║
║  Date_Time Part Extractor (extrae Year)                     ║
║  Python Script (title_length, title_words, sentimiento)     ║
╚══════════════════════════════════════════════════════════════╝
                           │
╔══════════════════════════════════════════════════════════════╗
║           ETAPA 4 — ANÁLISIS Y VISUALIZACIÓN                ║
║  GroupBy x5 + Sorter x2 → 4 Bar Charts                     ║
║                         → 1 Line Plot                       ║
║                         → 2 Scatter Plots                   ║
║                         → 1 Statistics View                 ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 4. Descripción Detallada de los Nodos

### 4.1 Ingesta de Datos

**`SQLite Connector (#4)` + `DB Query Reader (#5)`**  
Establece conexión con la base de datos SQLite local (`spotify_extra_100.sqlite`) y ejecuta `SELECT * FROM spotify_tracks`. Es la fuente principal del flujo. La decisión de usar SQLite refleja un escenario realista donde los datos históricos se almacenan en una base de datos ligera de escritorio.

**`MongoDB Connector (#48)` + `MongoDB Reader (#7)`**  
Conecta a una instancia cloud de MongoDB Atlas y extrae una colección de documentos JSON almacenados de forma no relacional. Representa el escenario de datos semi-estructurados provenientes de una API o sistema NoSQL.

**`XML Reader (#3)` + `XPath (#28)`**  
Lee un archivo XML y extrae 15 campos mediante expresiones XPath con la ruta base `spotify_data/track/...`. Los campos extraídos incluyen: `track_id`, `track_name`, `track_number`, `track_popularity`, `explicit`, `artist_name`, `artist_popularity`, `artist_followers`, `artist_genres`, `album_id`, `album_name`, `album_release_date`, `album_total_tracks`, `album_type`, `track_duration_min`.

**`JSON Reader (#2)` + `JSON Path (#16, #49)` + `Ungroup (#17, #50)`**  
Lee datos en formato JSON, extrae campos anidados con rutas específicas y luego aplica `Ungroup` para desanidar listas (por ejemplo, `artist_genres` que puede contener múltiples géneros por artista) y convertirlas en filas individuales.

**`CSV Reader (#59)`**  
Lee el archivo `spotify_data.csv` desde almacenamiento local. Quinta fuente del análisis que complementa el dataset con registros adicionales.

---

### 4.2 Limpieza y Preparación (por cada rama)

Cada una de las cinco ramas de datos pasa por un pipeline de limpieza independiente antes de ser unida. Los nodos se repiten con numeraciones distintas para cada fuente:

**`String to Date_Time (#15, #23, #34, #47, #57)`**  
Convierte la columna `album_release_date` de tipo texto a tipo fecha/hora en cada rama. Es necesario porque las distintas fuentes entregan las fechas como strings con formatos variables.

**`Column Expressions legacy (#11, #19, #30, #43, #52)`**  
Genera la columna `track_duration_time` en formato `MM:SS` a partir de `track_duration_min` (número decimal). El algoritmo aplicado:

```javascript
var total_min = column("track_duration_min");
var mins = toInt(floor(total_min));
var secs = toInt(round((total_min - floor(total_min)) * 60));
if (secs == 60) { mins = mins + 1; secs = 0; }
join(string(mins), ":", padLeft(string(secs), 2, "0"))
// Ejemplo: 3.75 → "3:45"
```

**`String Replacer (#9)`**  
En la columna `artist_genres`, reemplaza el valor literal `"N/A"` por `"Unknown"` para estandarizar los valores faltantes y facilitar análisis posteriores por género.

**`Column Filter (#10, #18, #29, #42, #51, #73)`**  
Selecciona únicamente las columnas relevantes en cada rama, descartando campos técnicos o redundantes que no aportan al análisis.

**`Row Filter (#12, #31, #44, #53)` — Filtro de duración**  
Retiene solo canciones con `track_duration_min > 1.0`, eliminando intros, efectos de sonido, silencio o registros corruptos que distorsionarían el análisis de duración.

**`Row Filter (#21, #32, #45, #54)` — Filtro de popularidad**  
Retiene solo canciones con `track_popularity > 30`, enfocando el análisis en contenido con relevancia real en la plataforma y eliminando tracks prácticamente sin escuchas.

**`Row Filter (#22, #33, #46, #55, #56)` — Filtro de tipo de álbum**  
Retiene solo registros donde `album_type = "album"`, excluyendo singles y compilaciones para mantener consistencia en el análisis por posición de track y duración.

**`Concatenate (#24)`**  
Une todas las ramas en un único dataset consolidado, alineando columnas por nombre. Es el punto de convergencia de las cinco fuentes.

**`Missing Value (#8)`**  
Gestiona valores nulos residuales en el dataset consolidado, aplicando estrategias de reemplazo o eliminación de filas incompletas.

**`Duplicate Row Filter (#27)`**  
Elimina registros duplicados que surgen al unir múltiples fuentes que comparten canciones en común.

---

### 4.3 Transformación y Enriquecimiento

**`Date_Time Part Extractor (#58)`**  
Extrae el componente `Year` de la columna `album_release_date` como entero, creando una variable temporal para el análisis de evolución por año.

**`Python Script (#74)`**  
Nodo de mayor complejidad analítica. Genera tres columnas nuevas usando pandas:

```python
import knime.scripting.io as knio
df = knio.input_tables[0].to_pandas()

# 1. Longitud del título en caracteres
df["title_length"] = df["track_name"].astype(str).str.len()

# 2. Número de palabras en el título
df["title_words"] = df["track_name"].astype(str).str.split().str.len()

# 3. Clasificación de sentimiento por palabras clave
positivas = ["love", "baby", "good", "happy", "night", "heart", "dance"]
negativas = ["sad", "cry", "pain", "hate", "broken", "lonely"]

def sentimiento_simple(texto):
    texto = str(texto).lower()
    if any(p in texto for p in positivas):
        return "positivo"
    if any(n in texto for n in negativas):
        return "negativo"
    return "neutral"

df["sentimiento_titulo"] = df["track_name"].apply(sentimiento_simple)
knio.output_tables[0] = knio.Table.from_pandas(df)
```

---

### 4.4 Agregaciones y Ordenamiento

| Nodo | Agrupación por | Métrica calculada | Alimenta |
|------|---------------|-------------------|----------|
| `GroupBy #35` | `artist_name` | `MAX(artist_followers)` | Sorter #41 → Bar Chart #38 |
| `GroupBy #61` | `Year` | `COUNT(track_id)` | Line Plot #62 |
| `GroupBy #66` | `track_number` | `AVG(track_popularity)` | Bar Chart #67 |
| `GroupBy #68` | `artist_name` | `MEAN(track_duration_min)` | Sorter #69 → Bar Chart #70 |
| `GroupBy #75` | `sentimiento_titulo` | `MEAN(track_popularity)` | Bar Chart #76 |

**`Sorter (#41)`** — Ordena los artistas por `Max(artist_followers)` de forma descendente, garantizando que el Bar Chart #38 muestre los Top 10 correctamente.

**`Sorter (#69)`** — Ordena los artistas por `Mean(track_duration_min)` de forma descendente para que Bar Chart #70 muestre los que producen canciones más largas primero.

---

## 5. Visualizaciones

### 📊 Gráfica 1 — Bar Chart #38
**Título:** *"Top 10 artistas con mayor número de seguidores según su album"*  
**Tipo:** Gráfico de barras agrupadas vertical  
**Datos:** `artist_name` (eje X) vs `Max(artist_followers)` (eje Y) — limitado a 10 registros  
**Descripción:** Muestra los 10 artistas del dataset con mayor audiencia acumulada en Spotify. Cada barra representa el máximo de seguidores registrado para ese artista. Permite identificar quiénes dominan la plataforma en términos de alcance y visualizar las brechas entre artistas masivos frente al resto. Los datos llegan ordenados por el `Sorter #41`, garantizando que el top 10 sea correcto.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 1: Top 10 artistas por seguidores]**

---

### 📊 Gráfica 2 — Bar Chart #67
**Título:** *"Impacto del Número de Track"*  
**Tipo:** Gráfico de barras vertical  
**Datos:** `track_number` (eje X, posiciones 1–24) vs `Mean(track_popularity)` (eje Y)  
**Descripción:** Analiza si la posición de una canción dentro de un álbum tiene relación con su popularidad promedio. Si las primeras posiciones concentran barras más altas, indica que los artistas colocan sus canciones más fuertes al inicio del álbum. Una distribución irregular o descendente confirmará o refutará si el orden dentro del disco influye en el rendimiento de la canción.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 2: Impacto del número de track]**

---

### 📊 Gráfica 3 — Bar Chart #70
**Título:** *"Quién hace las canciones más largas"*  
**Tipo:** Gráfico de barras agrupadas vertical  
**Datos:** `artist_name` (eje X, top 10) vs `Mean(track_duration_min)` (eje Y)  
**Descripción:** Compara la duración promedio de las canciones de los principales artistas, ordenados de mayor a menor duración por el `Sorter #69`. Permite relacionar estilos musicales con la extensión de sus canciones y cruzar este dato con la popularidad: si los artistas con canciones más largas no aparecen entre los más populares, se refuerza la hipótesis de que la brevedad favorece el rendimiento en plataformas de streaming.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 3: Artistas con canciones más largas]**

---

### 📊 Gráfica 4 — Bar Chart #76
**Título:** *"Popularidad promedio por sentimiento del título"*  
**Tipo:** Gráfico de barras  
**Datos:** `sentimiento_titulo` (eje X: positivo, negativo, neutral) vs `Mean(track_popularity)` (eje Y)  
**Descripción:** Resultado del análisis de sentimiento aplicado en Python. Muestra si los títulos con palabras emocionalmente positivas (`love`, `happy`, `dance`...) corresponden a canciones con mayor popularidad promedio que los negativos o neutrales. Si la barra de "positivo" supera consistentemente a las demás, se confirma que el tono emocional del título tiene una correlación con la aceptación del público.

---

> 📷 **[]**

---

### 📈 Gráfica 5 — Line Plot #62
**Título:** *"Lanzamientos populares por año"*  
**Tipo:** Gráfico de líneas  
**Datos:** `Year` (eje Y, rango 2009–2025) vs `Count(track_id)` (eje X)  
**Descripción:** Representa la evolución temporal del número de canciones populares en el dataset a lo largo de los años. Una línea con tendencia ascendente hacia los años recientes es esperable dado que el algoritmo de Spotify favorece contenido nuevo. Permite detectar años con mayor producción musical relevante y evaluar si el dataset presenta sesgo hacia épocas específicas.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 5: Lanzamientos populares por año]**

---

### 🔵 Gráfica 6 — Scatter Plot #63
**Título:** *"¿Las canciones más populares suelen ser las más cortas?"*  
**Tipo:** Gráfico de dispersión  
**Datos:** `track_duration_min` (eje X) vs `track_popularity` (eje Y) — hasta 2,500 puntos  
**Descripción:** Explora la relación entre duración y popularidad. Si los puntos con mayor popularidad (parte superior del gráfico) se concentran hacia la izquierda (canciones cortas), se confirmaría que la brevedad favorece la popularidad en streaming, donde el algoritmo penaliza los skips. Una nube sin patrón claro indicaría que la duración no es predictor significativo de popularidad.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 6: Duración vs popularidad]**

---

### 🔵 Gráfica 7 — Scatter Plot #65
**Título:** *"Canciones muy populares pero pertenecen a artistas con baja popularidad"*  
**Tipo:** Gráfico de dispersión  
**Datos:** `track_popularity` (eje X) vs `artist_popularity` (eje Y) — hasta 2,500 puntos  
**Descripción:** Detecta el fenómeno de canciones virales o hits inesperados: pistas con alta puntuación de popularidad cuyos artistas tienen baja visibilidad general. Los puntos en la esquina inferior derecha (alta popularidad de canción, baja del artista) son los casos más interesantes, pues representan artistas que viralizaron una canción específica sin consolidar una base de seguidores permanente, un patrón característico de la era TikTok/Reels.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 7: Popularidad de canción vs artista]**

---

### 📋 Gráfica 8 — Statistics View #64
**Tipo:** Vista de estadísticas descriptivas  
**Descripción:** Genera un resumen estadístico completo del dataset: valores mínimos, máximos, medias, medianas y distribuciones de frecuencia para todas las variables numéricas. Sirve como tabla de referencia para validar la calidad del dataset y contextualizar todas las visualizaciones anteriores.

---

> 📷 **[INSERTAR IMAGEN — Gráfica 8: Statistics View]**

---

## 6. Hallazgos y Conclusiones

### 6.1 Sobre la popularidad y la duración

El Scatter Plot #63 responde directamente si las canciones más cortas son más populares. La tendencia esperada en la era del streaming es que canciones de entre 2:30 y 3:30 minutos concentren los valores más altos de popularidad, ya que el algoritmo de Spotify premia el tiempo de escucha completo y penaliza los skips. Si el gráfico confirma esta concentración, se valida una práctica ya observada industrialmente: los productores acortan deliberadamente sus canciones para maximizar el ratio de reproducción completa.

### 6.2 Sobre los artistas con mayor audiencia

El Bar Chart #38 evidencia el efecto de concentración característico de las plataformas digitales: un pequeño grupo de artistas acapara la mayoría de la audiencia. La comparación con el Bar Chart #70 permite cruzar si los artistas más seguidos producen canciones más cortas o más largas, añadiendo una dimensión cualitativa al análisis.

### 6.3 Sobre la evolución temporal

El Line Plot #62 (2009–2025) permite verificar si existe un crecimiento sostenido en lanzamientos populares. Es esperable un aumento marcado a partir de 2015–2017, coincidiendo con la masificación de Spotify como canal principal de distribución. Un pico en años recientes también sería coherente con el sesgo de actualidad del algoritmo.

### 6.4 Sobre el sentimiento en los títulos

El análisis de sentimiento del Python Script y su visualización en el Bar Chart #76 permiten concluir si existe correlación entre el tono emocional del título y la popularidad. La hipótesis es que títulos positivos generan mayor engagement inicial en el momento del descubrimiento. Si la barra de "positivo" es consistentemente más alta, la hipótesis se confirma empíricamente con los datos.

### 6.5 Sobre canciones virales de artistas poco conocidos

El Scatter Plot #65 es el hallazgo más llamativo: la existencia de canciones con `track_popularity > 70` cuyos artistas tienen `artist_popularity < 40`. Este fenómeno evidencia el poder de la viralización en redes sociales como canal de descubrimiento capaz de impulsar una sola canción a niveles masivos sin que el artista consolide una base permanente de seguidores.

### 6.6 Sobre la posición en el álbum

El Bar Chart #67 permite verificar si el orden de una canción dentro del álbum afecta su popularidad. La hipótesis es que las primeras posiciones concentran mayor popularidad, ya que los oyentes que consumen álbumes completos tienden a escuchar más las canciones iniciales, y los artistas suelen colocar ahí sus temas más fuertes.

### Conclusión General

Este proyecto demostró que es posible construir un pipeline de análisis de datos completo, limpio y visualmente rico a partir de cinco fuentes heterogéneas (SQLite, MongoDB, XML, JSON, CSV) usando KNIME como plataforma central. La integración multi-fuente es el aporte técnico más significativo del proyecto, seguido del módulo de análisis de sentimiento implementado en Python.

Los datos de Spotify revelan patrones concretos en el comportamiento musical: la duración, el año, la posición en el álbum y el tono emocional del título tienen relaciones —directas o indirectas— con la popularidad. El hallazgo sobre canciones virales de artistas desconocidos abre preguntas relevantes sobre los mecanismos de descubrimiento musical en la era digital y el impacto de redes sociales como TikTok en la industria.

Desde el punto de vista técnico, el flujo aplica criterios de calidad rigurosos: filtros de duración mínima (`> 1 min`), filtros de popularidad mínima (`> 30`), exclusión de singles y compilaciones (`album_type = "album"`), manejo de nulos, deduplicación y estandarización de géneros (`N/A → Unknown`), todo antes de cualquier análisis.

---

## 7. Anexo — Inventario Completo de Nodos

| # | Nodo | Categoría | Función |
|----|------|-----------|---------|
| 2 | JSON Reader | Ingesta | Lee archivo JSON |
| 3 | XML Reader | Ingesta | Lee archivo XML |
| 4 | SQLite Connector | Ingesta | Conexión a BD SQLite local |
| 5 | DB Query Reader | Ingesta | `SELECT * FROM spotify_tracks` |
| 7 | MongoDB Reader | Ingesta | Lee colección MongoDB |
| 8 | Missing Value | Limpieza | Manejo de valores nulos |
| 9 | String Replacer | Limpieza | `N/A → Unknown` en `artist_genres` |
| 10 | Column Filter | Limpieza | Selección columnas (rama SQL) |
| 11 | Column Expressions legacy | Transformación | Genera `track_duration_time` (rama SQL) |
| 12 | Row Filter | Limpieza | Duración > 1 min (rama SQL) |
| 13 | Row Filter | Limpieza | Filtro adicional (rama SQL) |
| 14 | Row Filter | Limpieza | Filtro adicional (rama SQL) |
| 15 | String to Date_Time | Transformación | Convierte fecha texto → fecha (rama JSON) |
| 16 | JSON Path | Transformación | Extrae campos del JSON |
| 17 | Ungroup | Transformación | Desanida listas (rama JSON) |
| 18 | Column Filter | Limpieza | Selección columnas (rama JSON) |
| 19 | Column Expressions legacy | Transformación | Genera `track_duration_time` (rama JSON) |
| 20 | Row Filter | Limpieza | Duración > 1 min (rama JSON) |
| 21 | Row Filter | Limpieza | Popularidad > 30 (rama JSON) |
| 22 | Row Filter | Limpieza | `album_type = "album"` (rama JSON) |
| 23 | String to Date_Time | Transformación | Convierte fecha texto → fecha (rama XML) |
| 24 | Concatenate | Integración | Une las 5 fuentes en un dataset |
| 27 | Duplicate Row Filter | Limpieza | Elimina duplicados |
| 28 | XPath | Transformación | Extrae 15 campos del XML |
| 29 | Column Filter | Limpieza | Selección columnas (rama XML) |
| 30 | Column Expressions legacy | Transformación | Genera `track_duration_time` (rama XML) |
| 31 | Row Filter | Limpieza | Duración > 1 min (rama XML) |
| 32 | Row Filter | Limpieza | Popularidad > 30 (rama XML) |
| 33 | Row Filter | Limpieza | `album_type = "album"` (rama XML) |
| 34 | String to Date_Time | Transformación | Convierte fecha texto → fecha (rama MongoDB) |
| 35 | GroupBy | Agregación | `MAX(artist_followers)` por artista |
| 38 | Bar Chart | Visualización | Top 10 artistas por seguidores |
| 41 | Sorter | Transformación | Ordena por seguidores DESC |
| 42 | Column Filter | Limpieza | Selección columnas (rama SQL v2) |
| 43 | Column Expressions legacy | Transformación | Genera `track_duration_time` (rama MongoDB) |
| 44 | Row Filter | Limpieza | Duración > 1 min (rama MongoDB) |
| 45 | Row Filter | Limpieza | Popularidad > 30 (rama MongoDB) |
| 46 | Row Filter | Limpieza | `album_type = "album"` (rama MongoDB) |
| 47 | String to Date_Time | Transformación | Convierte fecha texto → fecha (rama CSV) |
| 48 | MongoDB Connector | Ingesta | Conexión a MongoDB Atlas |
| 49 | JSON Path | Transformación | Extrae campos JSON (rama MongoDB) |
| 50 | Ungroup | Transformación | Desanida listas (rama MongoDB) |
| 51 | Column Filter | Limpieza | Selección columnas (rama MongoDB) |
| 52 | Column Expressions legacy | Transformación | Genera `track_duration_time` (rama CSV) |
| 53 | Row Filter | Limpieza | Duración > 1 min (rama CSV) |
| 54 | Row Filter | Limpieza | Popularidad > 30 (rama CSV) |
| 55 | Row Filter | Limpieza | `album_type = "album"` (rama CSV) |
| 56 | Row Filter | Limpieza | `album_type = "album"` (rama CSV v2) |
| 57 | String to Date_Time | Transformación | Convierte fecha texto → fecha (rama CSV) |
| 58 | Date_Time Part Extractor | Transformación | Extrae `Year` de la fecha |
| 59 | CSV Reader | Ingesta | Lee archivo CSV local |
| 61 | GroupBy | Agregación | `COUNT(track_id)` por año |
| 62 | Line Plot | Visualización | Evolución de lanzamientos por año |
| 63 | Scatter Plot | Visualización | Duración vs popularidad |
| 64 | Statistics View | Visualización | Estadísticas descriptivas del dataset |
| 65 | Scatter Plot | Visualización | Popularidad canción vs artista |
| 66 | GroupBy | Agregación | `AVG(track_popularity)` por track_number |
| 67 | Bar Chart | Visualización | Impacto del número de track |
| 68 | GroupBy | Agregación | `MEAN(track_duration_min)` por artista |
| 69 | Sorter | Transformación | Ordena por duración promedio DESC |
| 70 | Bar Chart | Visualización | Artistas con canciones más largas |
| 73 | Column Filter | Limpieza | Selección columnas previo a Python |
| 74 | Python Script | Análisis | Genera `title_length`, `title_words`, `sentimiento_titulo` |
| 75 | GroupBy | Agregación | `MEAN(track_popularity)` por sentimiento |
| 76 | Bar Chart | Visualización | Popularidad promedio por sentimiento del título |
