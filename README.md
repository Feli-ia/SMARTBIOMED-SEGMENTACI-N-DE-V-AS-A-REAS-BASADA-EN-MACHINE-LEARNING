# SMARTBIOMED-SEGMENTACI-N-DE-V-AS-A-REAS-BASADA-EN-MACHINE-LEARNING

# 🫁 SMARTBIOMED: Segmentación de Vías Aéreas Basada en Machine Learning

> **"No se trata de imitar al experto, sino de modelar la anatomía."**

Este repositorio contiene la implementación completa del proyecto **SMARTBIOMED**, un sistema híbrido para la segmentación automática de vías aéreas en tomografía computarizada (TC), orientado a la cuantificación robusta del **Total Airway Count (TAC)** —un biomarcador estructural clave en la detección temprana de enfermedades pulmonares obstructivas como la EPOC y el asma.

El enfoque combina técnicas clásicas de procesamiento de imágenes con un modelo Swin UNETR entrenado exclusivamente con **supervisión sintética**, evitando por completo la dependencia de anotaciones manuales. El resultado es un pipeline reproducible, eficiente y clínicamente relevante.

---

## 📋 Tabla de Contenidos

- [1. Problema y Solución Propuesta](#1-problema-y-solución-propuesta)
- [2. Metodologías](#2-metodologías)
- [3. Implementación](#3-implementación)
- [4. Resultados](#4-resultados)
- [5. Glosario](#5-glosario)
- [6. Anexos](#6-anexos)

---

## 1. Problema y Solución Propuesta

La cuantificación fiable del **Total Airway Count (TAC)** —número total de ramas bronquiales visibles en TC— es un desafío crítico en la evaluación temprana de enfermedades pulmonares obstructivas. Los métodos actuales enfrentan dos limitaciones fundamentales:

1. **Dependencia de segmentaciones manuales**:  
   - Requieren **2–15 horas por caso**.  
   - Alta **variabilidad interobservador**.  
   - Impracticable para cribado masivo o seguimiento longitudinal.

2. **Fragilidad de métodos automáticos**:  
   - Subestiman sistemáticamente ramas distales (≥G4).  
   - Optimizan métricas voxel-wise (ej. Dice) que **no reflejan integridad topológica**.  
   - Sufren de **filtración** (confunden parénquima/vasos con vías aéreas).

Además, la resolución física de la TC clínica (≈0.7 mm) limita la visualización de bronquiolos finos (<2 mm), exacerbando la subestimación.

### Solución propuesta

Proponemos un **enfoque híbrido no supervisado** con tres pilares:

**Pipeline tubular robusto**:  
Crecimiento direccional desde la tráquea (BFS 26-conectado) con umbrales fisiológicos.

**Preprocesamiento minimalista**:  
Combinación óptima: **HU Clipping + Padding** (sin normalización ni suavizado innecesarios).

**Supervisión sintética**:  
Entrenamiento del Swin UNETR con labels generados automáticamente, eliminando dependencia humana.

**Resultado clave**:  
- **TAC = 50.0 ± 5.2 ramas** (vs. 47 ± 5 del estado del arte).  
- **Preservación hasta G4** con tasa de filtración <2%.  
- **Reproducible en <10 minutos por caso**.

---

## 2. Metodologías

### Marco general
Metodología experimental comparativa orientada a la segmentación automática de vías aéreas mediante técnicas clásicas y aprendizaje profundo.

### Materiales
- **Dataset**: 150 volúmenes TC del **ATM’22 Challenge** (sanos, EPOC, COVID-19).  
- **Resolución**: Isotrópica promedio de **0.7 mm**.  
- **Gold standard**: Segmentaciones manuales (TAC = 159, TACg peak = G5(40)).

### Tecnologías
- **Lenguaje**: Python 3.9  
- **Librerías**: `MONAI`, `SimpleITK`, `scikit-image`, `NumPy`, `SciPy`  
- **Hardware**: GPU NVIDIA RTX 3090 (24 GB VRAM)

### Métricas de evaluación
| Métrica | Descripción |
|--------|-------------|
| **TAC** | Número total de ramas conectadas |
| **TACg peak** | Generación con mayor número de ramas terminales |
| **Dice (DSC)** | Superposición espacial con gold standard |
| **Tasa de filtración** | % de casos con segmentación errónea de parénquima/vasos |

---

## 3. Implementación

El flujo de trabajo se organiza en **11 notebooks Jupyter**, divididos en tres fases:

### Fase 1: Preprocesamiento individual (`01–04`)
- `01-HU_clipping.ipynb`: Recorte HU a [-1024, 600]  
- `02-Padding_32.ipynb`: Relleno simétrico a múltiplos de 32  
- `03-Normalizacion.ipynb`: Normalización Min-Max a [0, 1]  
- `04-Gaussiano.ipynb`: Suavizado gaussiano (σ = 0.8)

### Fase 2: Pipeline tubular (`05–09`)
- `05-ROI.ipynb` y `06-Full_ROI.ipynb`: Generación de ROI pulmonar  
- `07-Preprocessing.ipynb` y `08-Full_Preprocessing.ipynb`: Evaluación de 15 combinaciones  
- `09-Procesamiento.ipynb`: Pipeline BFS completo → genera `ATM_XXX_hybrid_prediction.nii.gz`

### Fase 3: Análisis y modelo (`10–11`)
- `10-TAC.ipynb`: Esqueletización 3D + cálculo de TAC/TACg  
- `11-Modelo.ipynb`: Entrenamiento Swin UNETR con MONAI

### Problemas y soluciones
- **Filtración en enfisema**: Reducida de 15% a <2% con HU Clipping + Padding.  
- **Sobreajuste en Swin UNETR**: Mitigado con early stopping (30 épocas, 100 volúmenes).  
- **Detección de tráquea**: Optimizada con enfoque multi-eje (axial/coronal/sagital).

---

## 4. Resultados

### 4.1 Resultados cuantitativos

| Método | Dice | TAC | TACg peak |
|--------|------|-----|-----------|
| **Label Manual** | — | 159 | G5 (40) |
| **3D Slicer** | 0.78 | 47 ± 5 | G4 (14 ± 3) |
| **Label BFS** | 0.83 | 27.3 ± 22.3 | G4 (10.9 ± 7.2) |
| **HU Clipping + Padding** | 0.82 | **50.0 ± 5.2** | **G4 (11.0 ± 2.1)** |
| **Swin UNETR** | 0.70 | 25 | G3 (8) |

🔑 **Hallazgos clave**:  
- La mejor combinación **supera al estado del arte en TAC (+6.4%)** y **reduce filtración de 15% a <2%**.  
- El Swin UNETR, pese a su arquitectura avanzada, **subestima severamente ramas distales** (TAC=25 vs 50).

### 4.2 Validación cualitativa

![Figura 6: Comparación visual](Imagenes/output.png)

**Interpretación**:  
- **(A) Gold Standard**: Arquitectura completa hasta G5.  
- **(B) 3D Slicer**: Poda artificial en ramas periféricas.  
- **(C) HU Clipping + Padding**: Continuidad preservada hasta G4.  
- **(D) Swin UNETR**: Fragmentación severa, sin ramas distales.

### 4.3 Discusión crítica
- **Validación interna**: El preprocesamiento robusto es **más determinante que la complejidad del modelo**.  
- **Validación externa**: Resultados alineados con el ATM’22 Challenge: **integridad topológica > Dice**.  
- **Limitaciones**:  
  - TAC obtenido (50) aún lejos del gold standard (159) → límite físico de la TC.  
  - Swin UNETR limitado por datos escasos (100 volúmenes) y supervisión sintética.  

> **Conclusión metodológica**: Un método clásico bien diseñado supera a modelos profundos complejos cuando se prioriza la fidelidad anatómica.

---

## 5. Glosario

| Término | Definición |
|--------|------------|
| **BFS** | Búsqueda en amplitud para propagación tubular desde la tráquea (G0) |
| **Dice (DSC)** | Métrica de superposición: \( \text{DSC} = \frac{2\|X \cap Y\|}{\|X\| + \|Y\|} \) |
| **Filtración** | Error donde parénquima/vasos se confunden con vías aéreas |
| **HU** | Unidades Hounsfield (escala de densidades en TC; aire = -1024 HU) |
| **Padding** | Relleno simétrico con aire (-1024 HU) para garantizar bordes regulares |
| **ROI** | Región de interés (tejido pulmonar aislado del mediastino) |
| **Swin UNETR** | Arquitectura de segmentación 3D basada en transformers |
| **TAC** | Total Airway Count: número total de ramas bronquiales conectadas |
| **TACg** | Distribución del TAC por generación bronquial (G0 = tráquea) |
| **TC** | Tomografía computarizada |

---

## 6. Anexos


### Dataset
Los datos utilizados provienen del **ATM’22 Challenge**:  
- [TrainBatch1](https://doi.org/10.5281/zenodo.6370401)  
- [TrainBatch2](https://doi.org/10.5281/zenodo.6370402)

### Requisitos
```txt
monai==1.3.0
torch==2.1.0+cu118
SimpleITK==2.3.1
scikit-image==0.22.0
