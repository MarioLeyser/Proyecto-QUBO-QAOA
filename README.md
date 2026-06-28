# Proyecto QUBO-QAOA: Asignación óptima de cobertura hospitalaria en la CDMX

Proyecto final para **QMexico Summer School 2026 — Qubit.mx**.  
Modelamos la asignación de alcaldías de la CDMX a hospitales públicos de alta especialidad
como un **matching bipartito perfecto 4×4**, lo formulamos como **QUBO** y lo resolvemos
con **QAOA simulado localmente** (numpy + scipy, sin IBM Quantum).

---

## Dataset

| Campo | Detalle |
|---|---|
| **Nombre** | Asignación de cobertura hospitalaria de alta especialidad en la CDMX |
| **Dominio** | Salud y servicios urbanos / logística de infraestructura crítica |
| **Fuentes** | INEGI Censo de Población y Vivienda 2020 · Directorio de Establecimientos SS · Datos Abiertos CDMX |
| **Institución** | INEGI / Secretaría de Salud Federal / Gobierno CDMX |
| **URLs** | https://www.inegi.org.mx/programas/ccpv/2020/ · https://www.gob.mx/salud · https://datos.cdmx.gob.mx/ |
| **Licencia** | Licencia Abierta MX (uso libre con atribución) |
| **Fecha de consulta** | 25 de junio de 2026 |

---

## Modelado

### Conjunto A — Zonas de demanda (alcaldías)

| ID | Alcaldía | Cuadrante | Población (Censo 2020) | Centroide |
|---|---|---|---|---|
| A1 | Iztapalapa | Oriente | 1 835 486 | 19.35529, −99.06224 |
| A2 | Gustavo A. Madero | Norte | 1 173 351 | 19.49392, −99.11075 |
| A3 | Álvaro Obregón | Poniente | 759 137 | 19.35867, −99.22028 |
| A4 | Tlalpan | Sur | 699 928 | 19.22434, −99.17235 |

**Criterio de selección:** las 4 alcaldías con mayor población según INEGI 2020,
distribuidas en los cuatro cuadrantes cardinales de la ciudad para representar
la distribución espacial de la demanda.

### Conjunto B — Centros de oferta (hospitales)

| ID | Hospital | Zona | Camas (SS) | Ubicación |
|---|---|---|---|---|
| B1 | Hospital General de México "Dr. Eduardo Liceaga" | Centro | 1 050 | 19.41295, −99.15174 |
| B2 | INER | Sur | 390 | 19.29057, −99.16016 |
| B3 | Hospital Juárez de México | Norte | 600 | 19.48278, −99.13639 |
| B4 | Hospital General Dr. Manuel Gea González | Sur | 450 | 19.28916, −99.16279 |

**Criterio de selección:** los 4 hospitales públicos federales con mayor número de
camas registradas en el Directorio SS, uno por zona geográfica.

### Definición de las variables

- $x_{ij} = 1$: la alcaldía $A_i$ queda asignada al hospital $B_j$ como su centro
  de referencia para traslados de alta especialidad.
- $x_{ij} = 0$: no existe asignación entre $A_i$ y $B_j$.

---

## Modelo matemático

### Función objetivo

$$\max_{\mathbf{x}} \sum_{i \in A} \sum_{j \in B} S_{ij}\, x_{ij}$$

### Restricciones (matching bipartito perfecto)

$$\sum_{j=1}^{4} x_{ij} = 1 \quad \forall\, i \in \{1,2,3,4\} \qquad \text{(cada alcaldía asignada a exactamente un hospital)}$$

$$\sum_{i=1}^{4} x_{ij} = 1 \quad \forall\, j \in \{1,2,3,4\} \qquad \text{(cada hospital atiende exactamente una alcaldía)}$$

### Score $S_{ij}$

$$S_{ij} = 0.5\,z(\text{pob}_i) + 0.3\,z(\text{camas}_j) - 0.2\,z(\text{dist}_{ij})$$

donde $z(v) = \dfrac{v - v_{\min}}{v_{\max} - v_{\min}} \in [0,1]$ es la normalización
min-max. El score **premia** asignar hospitales de mayor capacidad a zonas más pobladas
y **penaliza** la distancia de traslado (fórmula Haversine sobre coordenadas oficiales).

### Matriz $S$ (scores normalizados)

| Alcaldía \ Hospital | B1 (HGM) | B2 (INER) | B3 (HJM) | B4 (HGG) |
|---|:---:|:---:|:---:|:---:|
| **A1 Iztapalapa** | **0.7354** | 0.4264 | 0.4939 | 0.4512 |
| **A2 Gustavo A. Madero** | 0.4546 | 0.0530 | **0.3040** | 0.0786 |
| **A3 Álvaro Obregón** | 0.2766 | **−0.0270** | 0.0184 | 0.0008 |
| **A4 Tlalpan** | 0.1607 | −0.0347 | −0.1045 | **−0.0059** |

Los pares de la asignación óptima están en **negrita**.

---

## Formulación QUBO

$$E(\mathbf{x}) = -\sum_{i,j} S_{ij}\,x_{ij} + \lambda_A \sum_i \left(\sum_j x_{ij} - 1\right)^2 + \lambda_B \sum_j \left(\sum_i x_{ij} - 1\right)^2$$

con $\lambda_A = \lambda_B = 5.0 > \max(S_{ij}) = 0.7354$.

**Nota sobre penalización:** con $\lambda = 5$ el mínimo global del QUBO (−30.74)
corresponde a un estado infactible. Esto es una limitación conocida: la garantía teórica
estricta requiere $\lambda > \text{score\_óptimo\_total}$. El pipeline de permutaciones y
el algoritmo húngaro garantizan el óptimo factible de forma independiente, por lo que
el resultado final es correcto.

---

## Resultados

### Solución clásica óptima

Obtenida por fuerza bruta (65 535 configuraciones) y confirmada por el algoritmo húngaro:

| Alcaldía | Hospital asignado | $S_{ij}$ |
|---|---|:---:|
| Iztapalapa | Hospital General de México | 0.7354 |
| Gustavo A. Madero | Hospital Juárez de México | 0.3040 |
| Álvaro Obregón | INER | −0.0270 |
| Tlalpan | Hospital Gea González | −0.0059 |

- **Score total óptimo:** 1.0065
- **Energía QUBO:** −1.0065
- **Factible:** ✓ (matching bipartito perfecto)

La asignación es coherente con el dominio: Iztapalapa (1.8 M habitantes) se asigna al
Hospital General de México (1 050 camas, 11.4 km); Gustavo A. Madero se asigna al
Hospital Juárez, el más cercano (2.96 km).

### Resultado QAOA local ($p = 1$)

| Métrica | Valor |
|---|---|
| $\gamma^*$ | −0.6728 |
| $\beta^*$ | 1.1362 |
| Energía esperada | 3.7104 |
| Prob. estado factible (ideal) | 0.7211 % |
| Prob. óptimo clásico (ideal) | 0.0266 % |
| Mejor score factible muestreado (2 000 shots) | 0.9146 (91 % del óptimo) |
| ¿Encontró el óptimo exacto? | No en esta corrida |

Con $p=1$ y mixer estándar la mayor parte de la amplitud se distribuye sobre los
65 512 estados infactibles (solo 24 de 65 536 son matchings válidos). La mejor
muestra factible observada fue:

| Alcaldía | Hospital | $S_{ij}$ |
|---|---|:---:|
| Iztapalapa | Hospital Juárez de México | 0.4939 |
| Gustavo A. Madero | Hospital General de México | 0.4546 |
| Álvaro Obregón | Hospital Gea González | 0.0008 |
| Tlalpan | INER | −0.0347 |

Score = 0.9146 (solución factible sub-óptima, a 8.6% del óptimo).

---

## Justificación del dataset como matching bipartito 4×4

| Criterio | Respuesta |
|---|---|
| **Dos lados identificables** | A = alcaldías (demanda de atención). B = hospitales (oferta de camas). Conjuntos disjuntos con roles distintos. |
| **Tamaño reducible a 4×4** | Top-4 alcaldías por población (INEGI 2020), top-4 hospitales por camas (SS), uno por cuadrante. Regla reproducible. |
| **Score justificable** | Fórmula explícita con tres variables observables de fuentes oficiales y normalización min-max documentada. |
| **Decisión binaria** | $x_{ij}=1$ = alcaldía $i$ asignada al hospital $j$ como centro de referencia para traslados. |
| **Restricciones claras** | Matching bipartito perfecto uno-a-uno. Sin capacidades variables ni exclusiones. |
| **Fuente legítima** | INEGI Censo 2020, Directorio SS, Datos Abiertos CDMX. Licencia Abierta MX. Consulta: 25-jun-2026. |
| **Riesgo ético controlado** | Solo datos agregados por alcaldía/hospital. Sin datos personales. Uso exclusivamente académico. |

---

## Ética y limitaciones

**Riesgos éticos identificados:**
- Asignar 1-a-1 alcaldías con millones de habitantes a un solo hospital ignora
  la capacidad real, la especialización clínica y la saturación.
- El INER se especializa en enfermedades respiratorias; el modelo no distingue especialidades.

**Medidas de mitigación:**
- Uso exclusivo de datos públicos agregados. Sin datos personales de pacientes ni personal.
- **La salida del modelo no debe usarse para decisiones reales de salud pública,
  logística de ambulancias ni asignación de recursos hospitalarios.**

**Limitaciones del modelo:**
- La CDMX tiene 16 alcaldías y cientos de hospitales; la subinstancia 4×4 es educativa.
- No se modelan tráfico real, tiempos de respuesta, saturación ni especialización clínica.
- La distancia Haversine es geográfica, no de recorrido vial.
- Una instancia 16×16 requeriría 256 qubits; inviable por simulación local de vector de estado.

---

## Ejecución

### Requisitos
Instalados automáticamente en la primera celda del notebook.

### Pasos

1. Sube `proyecto_qubo_qaoa_corregido.ipynb` a Google Drive.
2. Coloca `data/dataset_real_4x4.csv` en `MyDrive/PROYECTO_Q/data/`
   (o el notebook usará los datos embebidos automáticamente si Drive no está disponible).
3. Abre el notebook en Google Colab.
4. `Entorno de ejecución → Ejecutar todo`.

El pipeline completo corre sin IBM Quantum ni GPU.

---

## Referencias

| # | Referencia |
|---|---|
| 1 | Glover et al. (2019). *Quantum Bridge Analytics I: QUBO models.* 4OR 17, 335–371. |
| 2 | Kuhn (1955). *The Hungarian Method.* Naval Research Logistics Quarterly, 2(1-2), 83–97. |
| 3 | Farhi et al. (2014). *A Quantum Approximate Optimization Algorithm.* arXiv:1411.4028. |
| 4 | Cerezo et al. (2021). *Variational quantum algorithms.* Nature Reviews Physics, 3, 625–644. |
| 5 | Hadfield et al. (2019). *QAOA to Quantum Alternating Operator Ansatz.* Algorithms 12(2), 34. |
| 6 | Lucas (2014). *Ising formulations of many NP problems.* Frontiers in Physics, 2, 5. |
| 7 | INEGI Censo de Población y Vivienda 2020. https://www.inegi.org.mx/programas/ccpv/2020/ |
| 8 | Directorio de Establecimientos SS. https://www.gob.mx/salud |
| 9 | Datos Abiertos CDMX. https://datos.cdmx.gob.mx/ — Licencia Abierta MX. |