# Predicción de Compuestos Antivirales contra el Dengue

Trabajo Práctico Final — Quimioinformática  
**Autores:** Andrés Lax · Agustina Sosa

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
- `n_estimators=5`, `random_state=42`
- Evaluación con matriz de confusión y recall

#### Random Forest con Morgan Fingerprints
- Fingerprints Morgan (radio=2, tamaño=2048 bits) generados con `rdFingerprintGenerator`
- Representación estructural densa de cada molécula
- Modelo seleccionado para la validación final

### 6. Validación en compuestos FDA
- Dataset obtenido desde ChEMBL (fase clínica máxima = 4, ~3.594 compuestos)
- Predicción con el modelo Random Forest + fingerprints
- Resultado: 15 compuestos (~0,5%) predichos como activos, incluyendo antihistamínicos (pirilamina, tripelenamina), tensioactivos (tonzonios) y antibióticos (cefiderocol)

---

## Resultados

| Modelo | Recall (Active) |
|---|---|
| KNN (k=3) | variable según k |
| Random Forest + descriptores | — |
| Random Forest + fingerprints | 0.059 |

**Matriz de confusión (Random Forest + fingerprints, test set):**

|  | Pred. Active | Pred. Inactive |
|---|---|---|
| **Real Active** | 4 (TP) | 64 (FN) |
| **Real Inactive** | 5 (FP) | 1975 (TN) |

El recall bajo refleja el desbalance severo de clases del dataset original, un problema clásico en cribado virtual de fármacos.

---

## Conclusiones

- Los modelos basados en fingerprints capturan patrones estructurales relevantes, pero son sensibles al desbalance de clases.
- El overfitting es un riesgo con pocos estimadores; se recomienda usar validación cruzada y ajuste de hiperparámetros.
- La validación sobre compuestos FDA muestra coherencia biológica: los hits incluyen compuestos con actividad antiparasitaria y antiviral conocida.
- Como trabajo futuro se propone balanceo de dataset (SMOTE u oversampling), más estimadores en el Random Forest, y exploración de modelos de grafo (GNN) para representación molecular más rica.

---

## Tecnologías utilizadas

- Python 3 (Google Colab)
- [RDKit](https://www.rdkit.org/) — quimioinformática y descriptores moleculares
- [scikit-learn](https://scikit-learn.org/) — modelos de ML
- [ChEMBL Web Resource Client](https://github.com/chembl/chembl_webresource_client) — descarga de fármacos FDA
- [PubChemPy](https://pubchempy.readthedocs.io/) — interfaz con PubChem
- pandas · numpy · matplotlib · seaborn

---

## Estructura del repositorio

```
bioinformatics/
└── Grupo_13.ipynb    # Notebook principal con análisis completo
```

---

## Referencias

- PubChem Bioassay AID 540333: https://pubchem.ncbi.nlm.nih.gov/bioassay/540333
- Ministerio de Salud Argentina — Boletín epidemiológico dengue 2025
- Lipinski, C.A. et al. (1997). Experimental and computational approaches to estimate solubility and permeability in drug discovery. *Advanced Drug Delivery Reviews*.
- Baell, J.B. & Holloway, G.A. (2010). New substructure filters for removal of pan assay interference compounds (PAINS). *J. Med. Chem.*
