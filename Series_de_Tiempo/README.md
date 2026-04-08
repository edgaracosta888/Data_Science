# Predicción de Series de Tiempo por Particionamiento Cíclico

**Proyecta valores futuros de una serie de tiempo descomponiéndola en sub-series cíclicas homólogas y ajustando una regresión logarítmica sobre cada una — sin modelos de Machine Learning, sin hiperparámetros, sin GPU.**

---

## La idea

Los métodos clásicos de forecasting (ARIMA, Prophet, LSTM) modelan la serie completa: tendencia, estacionalidad y ruido, todo a la vez. Cuando la estacionalidad es fuerte, estos modelos necesitan capturar patrones periódicos complejos, lo que incrementa el costo computacional y la cantidad de datos requeridos.

Este enfoque invierte la lógica: **en lugar de modelar la estacionalidad, la elimina.**

Se parte la serie en sub-series de períodos homólogos — por ejemplo, todos los eneros, todos los febreros, todos los martes — y se ajusta una curva independiente sobre cada una. Cada sub-serie, al estar compuesta solo por períodos equivalentes, pierde la componente estacional y expone únicamente la tendencia subyacente, que en muchas series reales resulta ser logarítmica.

```
Serie original (48 meses, 2005–2008)
│
├── Partición: Mes = 1  →  [Ene 2005, Ene 2006, Ene 2007, Ene 2008]  →  y = m·ln(x) + b  →  Ene 2009
├── Partición: Mes = 2  →  [Feb 2005, Feb 2006, Feb 2007, Feb 2008]  →  y = m·ln(x) + b  →  Feb 2009
├── ...
└── Partición: Mes = 12 →  [Dic 2005, Dic 2006, Dic 2007, Dic 2008]  →  y = m·ln(x) + b  →  Dic 2009
```

El resultado es una proyección completa del período futuro, construida a partir de 12 regresiones simples de 4 puntos cada una.

---

## ¿Por qué funciona?

La estacionalidad no desaparece del pronóstico: se preserva de forma implícita. Al agrupar períodos homólogos, cada sub-serie hereda el nivel propio de su estación (diciembre siempre vende más que febrero) pero evoluciona solo en función del tiempo. La curva logarítmica captura esa evolución con dos coeficientes (`m`, `b`) y el modelo resultante es:

$$\hat{y}_{\text{partición}} = m \cdot \ln(x) + b$$

donde `x` es el valor del campo temporal (el año, en este caso) y la partición define a qué sub-serie pertenece el dato.

### Ventajas frente a modelos convencionales

| Aspecto | Modelos clásicos / ML | Particionamiento cíclico |
|---------|----------------------|--------------------------|
| Complejidad computacional | Alta (optimización iterativa, grids de hiperparámetros) | Mínima (fórmulas cerradas por mínimos cuadrados) |
| Datos requeridos | Decenas a miles de registros | Funciona con tan solo 3–4 puntos por partición |
| Interpretabilidad | Baja en modelos de caja negra | Total: cada predicción es una función logarítmica explícita |
| Estacionalidad fuerte | Requiere componentes adicionales (Fourier, dummies) | Se neutraliza por diseño al particionar |
| Escalabilidad | Costo crece con la serie | Costo crece solo con el número de particiones |

---

## Arquitectura de funciones

```
┌──────────────────────────────────────── PIPELINE ────────────────────────────────────────┐
│                                                                                          │
│   df_partition(df, column, periodo)                                                      │
│       └── Filtra el DataFrame por un valor específico de la columna de partición.        │
│                                                                                          │
│   reg_log(df, campo_origen, campo_destino1, campo_destino2, x)                           │
│       ├── Calcula Z = ln(campo_origen) y las sumas cruzadas SSZZ, SSZY.                │
│       ├── Obtiene coeficientes m, b por mínimos cuadrados para cada campo destino.      │
│       ├── Predice ŷ = m·ln(x) + b para el valor futuro x.                              │
│       └── Retorna df con la fila predicha + listas de coeficientes (m_list, b_list).     │
│                                                                                          │
│   df_predict(df, campo_origen, campo_particionado, camp_pred_1, camp_pred_2, x)          │
│       ├── Itera sobre cada valor único del campo de partición.                           │
│       ├── Aplica df_partition + reg_log a cada sub-serie.                                │
│       ├── Ensambla df_new: DataFrame original + filas predichas.                         │
│       └── Ensambla df_reg: valores ajustados por la regresión para cada partición.       │
│                                                                                          │
│   ══════════════════════════════════════════════════════════════════════════════════════   │
│   df_predict(...)   ◀── función de alto nivel que orquesta todo el pipeline               │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### Detalle de cada función

| Función | Entrada | Salida | Rol |
|---------|---------|--------|-----|
| `df_partition(df, column, periodo)` | DataFrame, nombre de columna, valor del período | DataFrame filtrado | Aísla una sub-serie homóloga (e.g., todos los eneros). |
| `reg_log(df, campo_origen, campo_destino1, campo_destino2, x)` | DataFrame particionado, campo temporal, dos campos a predecir, valor futuro | `(df_result, m_list, b_list)` | Calcula regresión logarítmica por mínimos cuadrados y añade la fila con la predicción. |
| `df_predict(df, campo_origen, campo_particionado, camp_pred_1, camp_pred_2, x)` | DataFrame completo, campo temporal, campo de partición, dos campos objetivo, valor futuro | `(df_new, df_reg)` | Orquesta el particionamiento y la regresión para todas las sub-series. Retorna los datos originales con predicciones y los valores ajustados por la curva. |

---

## Ejemplo rápido

```python
import pandas as pd

df = pd.read_csv('Ventas_Company_H.csv').sort_values(by=['Anio', 'Mes']).reset_index(drop=True)

# Predecir el año 2009 particionando por mes
df_new, df_reg = df_predict(df, 'Anio', 'Mes', 'Numero_Total_Ventas', 'Valor_Total_Ventas', 2009)

# df_new: DataFrame original (2005–2008) + 12 filas predichas (2009, una por mes)
# df_reg: valores ajustados por la curva logarítmica para cada mes
```

---

## Visualizaciones incluidas

El notebook incluye tres tipos de gráficas:

1. **Serie completa con doble eje Y** — Número y Valor de ventas en miles, con años enteros en el eje X y grid habilitado.

2. **Partición individual con ajuste** — Datos históricos (azul), predicción (rojo) y curva de regresión logarítmica (verde) para un mes específico.

3. **Serie completa + proyección 2009** — Línea continua para históricos y línea punteada (mismo color) para las predicciones, con puntos rojos marcando cada valor proyectado.

---

## Métricas de ajuste

El coeficiente de determinación $R^2$ mide qué proporción de la variabilidad de los datos es explicada por la curva logarítmica ajustada:

$$R^2 = 1 - \frac{\sum_{i}(y_i - \hat{y}_i)^2}{\sum_{i}(y_i - \bar{y})^2}$$

Un $R^2$ cercano a 1 indica que la regresión logarítmica captura con precisión la tendencia de cada sub-serie. Dado que el particionamiento elimina la estacionalidad y deja expuesta una tendencia suave y monótona, los valores de $R^2$ por partición tienden a ser altos, confirmando que el modelo logarítmico es una elección adecuada para la dinámica subyacente de este tipo de series.

---

## Generalización

Aunque el ejemplo usa meses como campo de partición y años como campo origen, el método es genérico:

| Granularidad | Campo de partición | Campo origen | Predicción |
|-------------|-------------------|-------------|------------|
| Mensual | Mes (1–12) | Año | Siguiente año |
| Semanal | Día de la semana (Lun–Dom) | Semana | Siguiente semana |
| Diario | Hora del día (0–23) | Día | Siguiente día |
| Trimestral | Trimestre (Q1–Q4) | Año | Siguiente año |

La única condición es que exista un **patrón cíclico** en el campo de partición y una **tendencia sostenida** en el campo origen.

---

## Requisitos

```
pandas
numpy
matplotlib
```

---

## Estructura del proyecto

```
Series_de_Tiempo/
├── README.md
├── Predicccion_Series_de_Tiempo_Particionadas.ipynb   # Notebook con funciones, demo y gráficas
└── Ventas_Company_H.csv                               # Dataset: 48 registros mensuales (2005–2008)
```
