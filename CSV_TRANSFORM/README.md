# CSV ↔ Pixel Encryption

**Transforma archivos CSV en imágenes PNG encriptadas con doble clave y reconviértelas al DataFrame original sin pérdida de información.**

---

## Concepto

Cada celda de un archivo CSV es texto. Cada carácter de texto se puede representar como un byte (0–255) en codificación UTF-8. Y cada byte se puede mapear a un tono de gris único dentro de un gradiente de 256 colores.

Este proyecto explota esa equivalencia: convierte un DataFrame completo —encabezados y registros— en una matriz de píxeles donde cada píxel codifica exactamente un byte del contenido original. El resultado es una imagen PNG que, a simple vista, no revela nada del dato que contiene.

La seguridad se refuerza con **doble factor de encriptación**: dos semillas numéricas independientes (`n` y `m`) que aleatorizan, respectivamente, el orden de los bytes y el orden de los colores del gradiente. Sin ambas claves correctas, la decodificación produce basura.

```
CSV  ──▶  DataFrame  ──▶  Bytes UTF-8  ──▶  Píxeles (PNG)
                              │                    │
                         clave n              clave m
                      (aleatoriza bytes)  (aleatoriza colores)
```

---

## Cómo funciona

### Encriptación

```python
Encriptador(df, path, n=37, m=101)
```

| Paso | Descripción |
|------|-------------|
| 1 | Se genera la lista de bytes `[0, 1, ..., 255]` y se baraja con la semilla **`n`**. |
| 2 | Se genera un gradiente de 256 grises (de `#FFFFFF` a `#000000`) y se baraja con la semilla **`m`**. |
| 3 | Se construye un diccionario `{byte_aleatorio: color_aleatorio}` con la correspondencia 1 a 1. |
| 4 | Cada celda del DataFrame se codifica a bytes UTF-8 (soporta tildes, cirílico, emojis, etc.) separando campos con un delimitador interno. |
| 5 | Cada byte se sustituye por su color asignado, formando una fila de píxeles por cada registro. |
| 6 | Se ensambla una imagen RGBA y se guarda como `.png` junto con un archivo `_metadata.json` que almacena las longitudes reales de cada fila. |

### Desencriptación

```python
df_recuperado = cargar_df_desde_imagen(path, n=37, m=101)
```

El proceso inverso reconstruye el diccionario con las mismas semillas, lee cada píxel de la imagen, lo mapea de vuelta al byte original y decodifica el texto UTF-8 para devolver un `pd.DataFrame` idéntico al de entrada.

---

## Arquitectura de funciones

El pipeline se compone de funciones modulares que se encadenan en dos operaciones principales:

```
┌─────────────────────────────────── ENCRIPTACIÓN ───────────────────────────────────┐
│                                                                                    │
│   ext_columns(df)  ──┐                                                             │
│                      ├──▶  split_cols_rows(df)  ──▶  encript_df(df, n, m)          │
│   ext_rows(df)     ──┘                                      │                      │
│                                                              ▼                      │
│                                                        plot_df(C, R)               │
│                                                              │                      │
│                                                              ▼                      │
│                                                  guardar_imagen_y_metadata(...)     │
│                                                                                    │
│   ═══════════════════════════════════════════════════════════════════════════════   │
│   Encriptador(df, path, n, m)   ◀── función de alto nivel que orquesta todo        │
└────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────── DESENCRIPTACIÓN ────────────────────────────────┐
│                                                                                    │
│   cargar_df_desde_imagen(path, n, m)                                               │
│       ├── Lee imagen PNG + metadata JSON                                           │
│       ├── Reconstruye diccionario con semillas (n, m)                              │
│       ├── Mapea cada píxel RGB → byte → texto UTF-8                                │
│       └── Retorna pd.DataFrame con inferencia de tipos numéricos                   │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### Detalle de cada función

| Función | Entrada | Salida | Rol |
|---------|---------|--------|-----|
| `ext_columns(df)` | DataFrame | `list[str]` | Extrae los nombres de las columnas. |
| `ext_rows(df)` | DataFrame | `list[list[str]]` | Extrae cada fila como lista de strings. |
| `split_cols_rows(df)` | DataFrame | `(list[int], list[list[int]])` | Codifica encabezados y filas a secuencias de bytes UTF-8 con delimitador `&$&`. |
| `encript_df(df, n, m)` | DataFrame, semilla bytes, semilla colores | `(list[str], list[list[str]])` | Genera el diccionario byte→color con doble aleatorización y convierte cada byte a su color hex. |
| `plot_df(C, R)` | Listas de colores hex | `(np.ndarray RGBA, list[int])` | Ensambla la imagen RGBA y calcula las longitudes de cada fila. |
| `guardar_imagen_y_metadata(img, lengths, path)` | Imagen, longitudes, ruta | Archivos `.png` + `.json` | Persiste la imagen y los metadatos en disco. |
| `Encriptador(df, path, n, m)` | DataFrame, ruta, claves | Archivos en disco | **Función principal de encriptación.** Orquesta todo el pipeline. |
| `cargar_df_desde_imagen(path, n, m)` | Ruta, claves | DataFrame | **Función principal de desencriptación.** Reconstruye el DataFrame original. |

---

## Ejemplo rápido

```python
import pandas as pd

# Cargar datos
df = pd.read_csv('Ventas_Company_H.csv')

# Encriptar con claves n=37, m=101
Encriptador(df, 'VENTAS/', n=37, m=101)
# Genera: VENTAS.png + VENTAS_metadata.json

# Desencriptar con las mismas claves
df_recuperado = cargar_df_desde_imagen('VENTAS/', n=37, m=101)

# Verificar integridad
df.equals(df_recuperado)  # True
```

---

## Archivos generados

| Archivo | Contenido |
|---------|-----------|
| `*.png` | Imagen RGBA donde cada píxel es un byte encriptado del CSV. La primera fila de píxeles corresponde a los encabezados; las siguientes, a los registros. |
| `*_metadata.json` | Longitudes reales de cada fila (necesarias porque las filas se rellenan con transparencia hasta igualar la más larga). |

---

## Soporte de caracteres

La codificación opera a nivel de bytes UTF-8, lo que garantiza compatibilidad con:

- Caracteres latinos con tildes y diacríticos (`á`, `ñ`, `ü`)
- Alfabetos no latinos (cirílico, griego, árabe, CJK)
- Símbolos y emojis (`€`, `😀`, `→`)
- Cualquier carácter representable en Unicode

---

## Requisitos

```
pandas
numpy
matplotlib
Pillow
```

---

## Estructura del proyecto

```
CSV_TRANSFORM/
├── README.md
├── csv_transform.ipynb      # Notebook con todas las funciones y demo
├── Ventas_Company_H.csv     # Dataset de ejemplo
├── VENTAS.png               # Imagen encriptada (ejemplo generado)
└── VENTAS_metadata.json     # Metadatos de la imagen
```
