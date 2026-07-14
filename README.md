# Presupuesto de Ventas Tiendas propias:
# Enfoque Híbrido ML + Baseline

**Presupuesto de ventas para 2 tiendas propias a consumidor final. Una guía completa de cómo construir un modelo de pronóstico de ventas en producción: desde exploración de datos hasta validación y exportación. Dado que es un proyecto real de mi trabajo, todos los datos sensibles están anonimizados y factorizados para mantener confidencialidad.**

---
## 📋 Resumen Ejecutivo

Este proyecto demuestra cómo construir **un presupuesto de ventas que supera métodos tradicionales** combinando:
1. **Modelo estadístico** (factores estacionales + patrones por día de semana)
2. **Machine Learning** (Gradient Boosting con pérdida Poisson)
3. **Reconciliación híbrida** (lo mejor de ambos)

**Resultados clave (validados en mayo-junio 2026):**

| Métrica | Método anterior | Nuevo presupuesto | Mejora |
|---------|---|---|---|
| **Precisión diaria (por tienda)** | 27,0% error | 12,6% error | **↓ 53% mejor** |
| **Precisión mensual (por producto)** | 25,1% error | 23,1% error | ↓ 8% mejor |
| **Sesgo (sobre/subestimación)** | −10,5% | −2,2% | **↓ 79% mejor** |

El nuevo presupuesto captura **patrones reales de negocio** (fin de semana vs laboral, estacionalidad mensual, días especiales) que el método lineal anterior pasaba por alto.

---

## 🎯 El Problema

**El método anterior:**
- Calculaba el promedio de los últimos 2 meses
- Dividía ese promedio equitativamente entre todos los días del mes
- Asumía que todos los lunes son iguales a todos los miércoles

**¿Qué estaba mal?**
- Las ventas del sábado son **2-3 veces mayores** que las del martes, pero el presupuesto no lo sabía
- Septiembre (ciclo débil de invierno) es **14% más flojo** que mayo, pero el presupuesto trataba todos los meses igual
- El presupuesto sistemáticamente **subestimaba en −10,5%** en un período de 2 meses

**El costo:**
- El equipo de producción no sabía cuánto elaborar cada día → stockouts o desperdicio
- Compras hacía pedidos basados en un presupuesto plano → niveles de inventario incorrectos
- Las decisiones se tomaban sin entender la curva real de demanda

---

## 💡 La Solución: Modelo Híbrido de Presupuesto

### Tres Componentes

#### 1. **Nivel: Base Reciente** (¿Cuánto vende hoy cada producto?)

```
Base por producto = promedio de ventas, mayo-junio 2026, por tienda
```

- Captura el momentum actual del negocio (2026 vende ~19% más que 2025)
- Ancla en datos recientes, no en el pasado distante
- Resultado: **~##### unidades/mes de base**

#### 2. **Forma: Estacionalidad Mensual** (¿Cuáles son los meses fuertes/débiles?)

```
Factor estacional = ventas(mes en 2025) / promedio(mayo-junio 2025)
```

- Julio–agosto: ×1,03 (receso invernal, vacaciones escolares)
- Septiembre: ×0,86 (mes más débil del año)
- Octubre: ×0,98 (recuperación)
- Noviembre–diciembre: ×0,95–0,85 (cierre de año)

Se calcula **por familia de producto** para que productos nuevos hereden la estacionalidad de su categoría.

#### 3. **Timing: Distribución Diaria** (¿Cuándo dentro del mes?)

```
ventas_diaria = total_mensual × (predicción_modelo_para_este_día / Σ predicciones_mes)
```

**Modelo:** Gradient Boosting Regressor (pérdida Poisson)
- Entrenado con datos históricos de enero 2025 – junio 2026 (16 meses de datos diarios)
- 146.664 filas incluyendo ceros (53% de los días no hubo venta para un SKU dado. Dado que la demanda de algunos productos es intermintente, hubo que rellenar con ceros para evitar el sesgo positivo)
- Features principales:
  - `roll28`: media de los últimos 28 días (persistencia del nivel)
  - `dow_mean4`: media del mismo día de semana en las últimas 4 semanas (patrón semanal)
  - `dow`, `mes`, `dia_mes`, `feriado`: features de calendario

**¿Por qué pérdida Poisson?** La demanda es un conteo (enteros no negativos, muchos ceros). Poisson es el estándar para demanda retail; MSE permitiría predicciones negativas y maneja mal la escasez (el entrenamiento del modelo se realizó con un informe de venta con valores negativos por devolución, por eso fue necesario utilizarla)

---

## 📊 Metodología: Protocolo de Validación

Para probar que el modelo funciona **sin hacer trampa**, usamos una separación temporal adecuada:

1. **Datos disponibles:** enero 2025 – abril 2026
2. **Pronóstico:** mayo–junio 2026 (el modelo NO ve estos meses)
3. **Medición:** comparamos pronóstico vs ventas reales en mayo–junio 2026
4. **Comparación:** nuevo método vs método anterior sobre el mismo conjunto de prueba

**¿Por qué esto importa?**
- Evita overfitting al período de prueba
- Refleja la realidad en producción: "conozco datos hasta abril; pronostica mayo-junio"
- Comparación honesta: ambos métodos evaluados sobre el mismo futuro desconocido

**Resultados en el conjunto de validación:**

| Modelo | WMAPE SKU-día | WMAPE tienda-día | WMAPE SKU-mes | Sesgo |
|--------|---|---|---|---|
| Método anterior | 50,5% | 27,0% | 25,1% | −10,5% |
| Modelo A (Baseline) | 45,1% | 14,0% | 23,1% | −2,2% |
| **Híbrido (A + B)** | **44,1%** | **12,6%** | **23,1%** | **−2,2%** |

**Interpretación:**
- **A gana en totales mensuales** (el GBM solo ve 1 observación por mes, no puede aprender bien estacionalidad)
- **B gana en distribución diaria** (ML captura interacciones producto×día que el baseline promedia)
- **Híbrido combina ambos:** totales mensuales del A (17% mejor que el −10,5% anterior) + distribución diaria del B (10% mejor en precisión diaria)

---

## 🔍 Importancia de Features

¿Qué usa realmente el modelo?

```
dow_mean4:     0.886  ← "¿Cuánto vendió este producto en este día de semana el mes pasado?"
roll28:        0.697  ← "¿Cuál es el promedio reciente de este producto?"
dow:           0.118  ← Día de semana del calendario
FAMILIA:       0.106  ← Familia/categoría de producto
mes:           0.055  ← Mes del calendario (estacionalidad)
Tienda:        0.050  ← Cuál tienda
[otros]:       < 0.05
```

**Insight clave:** ~90% del valor del pronóstico viene de "qué vendió recientemente este producto" + "cuánto vende los sábados". Un pronóstico de demanda es, en esencia, **un promedio reciente inteligente**.

---

## 📁 Estructura del Proyecto

```
├── 01_carga_limpieza.ipynb          # Cargar y limpiar datos
├── 02_eda.ipynb                     # Explorar patrones
├── 03_modelo_baseline.ipynb         # Modelo estadístico
├── 04_modelo_gbm.ipynb              # Modelo ML (Gradient Boosting)
├── 05_presupuesto_final.ipynb       # Presupuesto final + exportación
├── Sales_Forecast_Anonymized.xlsx   # Archivo de presupuesto
├── environment.yml                   # Dependencias
└── README.md                         # Este archivo
```

---

## ⚙️ Configuración y Reproducción

### Instalar Dependencias

```bash
conda env create -f environment.yml
conda activate sales-forecast
```

O manualmente:
```bash
conda install pandas openpyxl scikit-learn matplotlib seaborn jupyter
```

### Ejecutar el Pipeline Completo
(los archivos ypynb fueron pasados a HTML exclusiva visualización)
```bash
jupyter lab 01_carga_limpieza.ipynb
# Ejecutar todas las celdas (Shift+Enter)
# → genera daily.pkl, maestro.pkl

jupyter lab 02_eda.ipynb
# Explora patrones, visualiza curva ABC, factores estacionales

jupyter lab 03_modelo_baseline.ipynb
# Construye modelo estadístico, reporta métricas de validación

jupyter lab 04_modelo_gbm.ipynb
# Entrena modelo ML, compara vs baseline, muestra importancia de features

jupyter lab 05_presupuesto_final.ipynb
# Genera presupuesto final y exporta Excel
```

### Archivos de Salida Esperados

- `daily.pkl`, `maestro.pkl` → datos limpios
- `Sales_Forecast_Final.xlsx` → presupuesto final (con números reales, confidencial)

---

## 🔐 Aviso de Anonimización

**Todos los valores en este repositorio han sido escalados por un factor de ### por confidencialidad.**

- Nombres de tiendas:  StoreA, StoreB
- Productos: Product001, etc.
- Cantidades: todas multiplicadas por ###

**¿Por qué escalar?** Los números reales de unidades son sensibles. El escalado preserva:
- ✓ Proporciones relativas (Producto A sigue siendo 80% del volumen)
- ✓ Patrones estacionales (julio sigue siendo 5% más fuerte que el promedio)
- ✓ Estructura del modelo (pesos, importancia de features, precisión del pronóstico)

**La metodología es completamente reproducible** con cualquier dataset; toda la lógica está en el código.

---

## 📈 Hallazgos Clave

1. **Los patrones semanales importan:** Las ventas del sábado son **~40% del volumen semanal**; martes es el mejor día laboral.
2. **La estacionalidad es real:** Julio–agosto son 3–5% más fuertes; septiembre es −14% más débil. El pronóstico lineal pasó esto por alto.
3. **El nivel reciente domina:** El promedio de los últimos 28 días explica ~70% de la demanda de mañana. Factores estacionales explican ~20%, efectos de calendario/familia ~10%.
4. **Concentración de mix:** Los 36 productos principales (18% del surtido) explican 80% del volumen (principio de Pareto).
5. **ML agrega valor incremental:** El baseline estadístico logra 48% de reducción de error. GBM agrega 10% más. El híbrido obtiene ambos sin complejidad excesiva.

---

## 🎓 Habilidades Demostradas

**Técnicas:**
- Ingeniería de datos (ETL, features causales sin fuga)
- Análisis exploratorio (descomposición estacional, curvas ABC)
- ML balanceado (Gradient Boosting, pérdida Poisson, validación temporal)
- Evaluación rigurosa (WMAPE, backtesting, análisis de sesgo)
- Producción (exportación a Excel, reconciliación, documentación)

**Negocio:**
- Delimitación del problema (por qué falló el método anterior)
- Hipótesis y pruebas (baseline vs ML)
- Elección de métricas (WMAPE y por qué importa)
- Comunicación honesta (tradeoffs, qué gana cada modelo)

**Profesionales:**
- Documentación (README, supuestos claros)
- Reproducibilidad (environment.yml, sin rutas hardcodeadas)
- Control de versiones (.gitignore, workflow limpio)
- Narrativa en notebooks (markdown + código + resultados)

---

## 📚 Referencias

- **Competencia M5 de Pronóstico** (Walmart, Kaggle): demanda retail es difícil; Gradient Boosting ganó
- **Regresión Poisson para Datos de Conteo:** estándar en modelado de demanda
- **Validación Temporal:** mejores prácticas de Rob Hyndman en forecasting
- **Métrica WMAPE:** estándar de la industria retail

---

**Última actualización:** julio 2026  
