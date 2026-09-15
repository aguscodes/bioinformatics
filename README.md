# Predicción de Compuestos Antivirales contra el Dengue

Trabajo Práctico Final 2.0 — Quimioinformática  
**Autores:** Agustina Sosa

---

## Descripción

El dengue es una enfermedad viral transmitida por el mosquito *Aedes aegypti* que afecta a millones de personas en regiones tropicales y subtropicales. En Argentina, durante 2025 se registraron más de 75.000 casos sospechosos, y a la fecha **no existe un antiviral aprobado** para tratar la infección.

Este proyecto aplica técnicas de quimioinformática y machine learning para identificar compuestos con potencial actividad antiviral contra el virus del dengue, entrenando modelos sobre datos de bioensayo de PubChem y validándolos frente a moléculas aprobadas por la FDA.

---

## Hipótesis y Objetivo

**Hipótesis:** Es posible entrenar un modelo de ML capaz de distinguir moléculas activas contra el dengue a partir de sus propiedades estructurales y fisicoquímicas.

**Objetivo:** Entrenar modelos de clasificación que reconozcan compuestos activos en el dataset de referencia, para luego aplicarlos como herramienta de *virtual screening* sobre fármacos aprobados por la FDA.

---

## Dataset

- **Fuente:** PubChem Bioassay [AID 540333](https://pubchem.ncbi.nlm.nih.gov/bioassay/540333)
- **Proyecto:** Broad Institute — ensayo de inhibición del efecto citopático (CPE) del virus del dengue
- **Variable objetivo:** `PUBCHEM_ACTIVITY_OUTCOME` (Active / Inactive)
- **Distribución:** 318 compuestos activos / 9.922 inactivos (ratio 1:31)

---

## Metodología

### 1. Limpieza de datos
- Eliminación de columnas vacías e irrelevantes
- Remoción de duplicados moleculares usando InChIKey (identificador estructural canónico)
- Descarte de filas sin etiqueta de actividad

### 2. Análisis exploratorio
- Visualización de distribuciones de propiedades fisicoquímicas (activos vs. inactivos)
- Cálculo de descriptores moleculares con RDKit

### 3. Descriptores moleculares calculados

| Descriptor | Descripción |
|---|---|
| `MolWt` | Peso molecular |
| `LogP` | Lipofilicidad |
| `NumHDonors` | Donores de puente H |
| `NumHAcceptors` | Aceptores de puente H |
| `TPSA` | Área de superficie polar topológica |
| `NumRotatableBonds` | Bonds rotables |
| `RingCount` | Cantidad de anillos |
| `atomos_aromaticos_proportion` | Proporción de átomos aromáticos |
| `non_Carbono_proportion` | Proporción de átomos no carbono |

### 4. Filtros fisicoquímicos
- **Regla de Lipinski (Ro5):** druglikeness para compuestos orales
- **Regla de los 3:** criterio para fragmentos de baja masa molecular
- **Filtro PAINS:** detección de compuestos con señales falsas en ensayos biológicos (PAINS A, B y C)

### 5. Modelos entrenados

#### KNN con TPSA y LogP
- Clasificador K-Nearest Neighbors con descriptores fisicoquímicos
- Evaluación de distintos valores de k (1, 3, 4, 5, 6, 9)
- Métrica principal: recall sobre la clase "Active"

#### Random Forest con descriptores
- 9 descriptores fisicoquímicos como features
- Evaluación con matriz de confusión y recall

#### Random Forest con Morgan Fingerprints
- Fingerprints Morgan (radio=2, tamaño=2048 bits) generados con `rdFingerprintGenerator`
- Representación estructural densa de cada molécula
- Modelo seleccionado para la validación final

### 6. Validación en compuestos FDA
- Dataset obtenido desde ChEMBL (fase clínica máxima = 4)
- Filtro de druglikeness aplicado mediante la Regla de Lipinski antes de la predicción
- Predicción con el modelo Random Forest + fingerprints

---

## Resultados

### Versión inicial

| Modelo | Recall train (Active) | Recall test (Active) |
|---|---|---|
| Random Forest + descriptores | 0.996 | 0.074 |
| Random Forest + fingerprints | 0.976 | 0.059 |

El recall bajo en test y el alto en train evidenciaban overfitting severo, producto del desbalance de clases (1:31) y la ausencia de regularización.

### Versión mejorada

Se aplicaron las siguientes modificaciones al modelo Random Forest + fingerprints:

- `class_weight='balanced'` para penalizar errores en la clase minoritaria
- Sobremuestreo sintético con SMOTE sobre el conjunto de entrenamiento
- Ajuste de `max_depth=7` seleccionado manualmente tras constatar que GridSearchCV con datos SMOTE sobreestimaba el rendimiento real en test
- `n_estimators=100`

| Modelo | Recall train (Active) | Recall test (Active) |
|---|---|---|
| Random Forest + fingerprints (mejorado) | 0.952 | 0.324 |

**Matriz de confusión (test set):**

|  | Pred. Active | Pred. Inactive |
|---|---|---|
| **Real Active** | 18 (TP) | 50 (FN) |
| **Real Inactive** | 254 (FP) | 1726 (TN) |

La mejora en recall (0.059 → 0.324) se obtiene al costo de mayor cantidad de falsos positivos, lo que refleja el trade-off inherente al desbalance de clases del dataset.

---

## Conclusiones

El modelo logra captar patrones estructurales asociados a la actividad contra el dengue, aunque el desbalance severo del dataset (1:31) limita su capacidad de generalización. Los Morgan fingerprints resultan una representación efectiva de la estructura molecular, pero la alta dimensionalidad (2048 bits) combinada con pocos ejemplos activos favorece el overfitting.

La incorporación de SMOTE, regularización por profundidad máxima y ajuste de hiperparámetros permitió mejorar el recall en test de 0.059 a 0.324. Sin embargo, la precisión permanece baja: el modelo tiende a predecir un número elevado de falsos positivos, lo que limita su utilidad directa como herramienta de screening. La optimización por F1 no produjo mejoras sustanciales respecto a la optimización por recall, y la profundidad óptima se mantuvo en `max_depth=7` en ambos casos.

La validación sobre compuestos aprobados por la FDA muestra resultados cualitativamente coherentes en algunos hits, pero la tasa de predicciones positivas (~42%) es demasiado alta para ser útil en la práctica, lo que refuerza que el problema central es el desbalance del dataset y no la elección del modelo.

Como líneas de trabajo futuro se propone:

- Incorporar bioensayos adicionales de dengue disponibles en PubChem para aumentar la cantidad de ejemplos activos
- Explorar modelos preentrenados sobre grandes corpus moleculares (ChemBERTa, MolBERT) que puedan aprovechar representaciones más ricas que los fingerprints
- Investigar arquitecturas de redes neuronales de grafos (GNN) que traten la molécula como grafo en lugar de vector de bits

---

## Tecnologías utilizadas

- Python 3 (Google Colab)
- [RDKit](https://www.rdkit.org/) — quimioinformática y descriptores moleculares
- [scikit-learn](https://scikit-learn.org/) — modelos de ML
- [imbalanced-learn](https://imbalanced-learn.org/) — SMOTE para balanceo de clases
- [ChEMBL Web Resource Client](https://github.com/chembl/chembl_webresource_client) — descarga de fármacos FDA
- [PubChemPy](https://pubchempy.readthedocs.io/) — interfaz con PubChem
- pandas · numpy · matplotlib · seaborn

---

## Estructura del repositorio

```
bioinformatics/
└── dengue_screening.ipynb    # Notebook principal con análisis completo
```

---

## Referencias

- PubChem Bioassay AID 540333: https://pubchem.ncbi.nlm.nih.gov/bioassay/540333
- Ministerio de Salud Argentina — Boletín epidemiológico dengue 2025
- Lipinski, C.A. et al. (1997). Experimental and computational approaches to estimate solubility and permeability in drug discovery. *Advanced Drug Delivery Reviews*.
- Baell, J.B. & Holloway, G.A. (2010). New substructure filters for removal of pan assay interference compounds (PAINS). *J. Med. Chem.*
- Chithrananda, S. et al. (2020). ChemBERTa: Large-Scale Self-Supervised Pretraining for Molecular Property Prediction. *arXiv*.
