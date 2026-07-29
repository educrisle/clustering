# Segmentación de Clientes para Campañas de Marketing

### Identificación de perfiles de cliente mediante Clustering no supervisado

---

## Contexto y problema de negocio

Las campañas de marketing masivo, dirigidas a toda la base de clientes por
igual, generan una ineficiencia silenciosa pero medible:

- Se invierte el mismo presupuesto en clientes de alto valor que en clientes
  que apenas compran, sin distinguir el retorno esperado de cada uno.
- No se sabe qué perfil de cliente es más receptivo a las campañas, así que
  el mensaje y el canal se eligen igual para todos.
- Sin una segmentación explícita, es imposible priorizar a quién proteger,
  a quién activar y a quién dejar de contactar.

La pregunta de negocio que motiva este proyecto: **¿existen grupos naturales
de clientes con comportamientos de compra distintos, y si es así, cómo debería
cambiar la estrategia de campaña para cada uno?**

---

## Objetivo

Construir una segmentación de clientes basada en su comportamiento real de
compra (ingreso, gasto, composición familiar, receptividad a campañas
anteriores) para priorizar presupuesto de marketing hacia los segmentos de
mayor retorno y rediseñar la estrategia para los de menor valor.

---

## Dataset

| Característica | Detalle |
|---|---|
| Registros | 2.212 clientes (2.240 originales, 1 outlier eliminado en `Income`) |
| Variables originales | 29 |
| Variables usadas en el clustering | 19, tras feature engineering y filtrado por coeficiente de variación |
| Valores nulos | 24 en `Income`, filas eliminadas (<1% del dataset) |
| Volumen de compra analizado | 1.343.294 € |
| Segmentos identificados | 4 |

**Variables clave construidas:**

| Variable | Tipo | Descripción |
|---|---|---|
| `Age` | int | Edad del cliente, calculada de forma dinámica desde `Year_Birth` |
| `TotalSpend` | float | Gasto total en las 6 categorías de producto (vinos, carne, pescado, frutas, dulces, oro) |
| `TotalPurchases` | int | Compras totales en los 4 canales (web, catálogo, tienda, ofertas) |
| `TotalAcceptedCmps` | int | Nº de campañas anteriores aceptadas, incluida la respuesta actual |
| `FamilySize` | int | Tamaño del hogar, derivado de `Marital_Status`, `Kidhome` y `Teenhome` |
| `HasPartner` | bool | Si el cliente tiene pareja registrada |
| `CustomerSince` | float | Antigüedad como cliente en años |

---

## Metodología

### Preprocesamiento

- Detección y eliminación del outlier de `Income` mediante la regla de la
  cerca (`Q3 + 3·IQR`), más conservadora que el estándar de 1.5·IQR para no
  penalizar ingresos altos legítimos.
- Feature engineering de variables agregadas (ver tabla anterior) y limpieza
  de categorías inconsistentes en `Marital_Status` (`"Absurd"`, `"YOLO"`).
- Codificación ordinal de `Education` (respeta el orden natural de los
  niveles educativos, a diferencia de un one-hot).
- Filtrado de variables por coeficiente de variación (CV < 0.10) para
  excluir columnas con poca capacidad discriminativa.
- Estandarización (`StandardScaler`) previa a cualquier técnica basada en
  distancia.

### Modelado

Se aplicó **PCA** (8 componentes, 82,9% de varianza explicada) antes del
clustering, decisión validada empíricamente comparando KMeans sobre el
espacio original escalado frente al espacio reducido: el espacio PCA
obtuvo mejores resultados en las tres métricas de validación.

**Pipeline de entrenamiento:**

```
StandardScaler → PCA (8 componentes) → KMeans (k=4)
                                     ↘ GaussianMixture (soft clustering)
```

- Selección de *k* combinando criterio estadístico (Elbow Method,
  Silhouette Score) y criterio de negocio: aunque k=2 maximiza el
  Silhouette, se eligió k=4 por ofrecer segmentos más accionables para
  campañas diferenciadas.
- `GaussianMixture` para obtener probabilidades de pertenencia (soft
  clustering) e identificar clientes con asignación ambigua, candidatos a
  campañas de activación.
- `RandomForest` entrenado sobre las variables originales para identificar
  qué características discriminan mejor entre segmentos (análisis
  descriptivo, no predictivo).

### Validación

- **Silhouette Score, Davies-Bouldin Index, Calinski-Harabasz Index** para
  evaluar compactación y separación de los clusters.
- **Estabilidad**: Adjusted Rand Index (ARI) comparando el modelo entrenado
  con 8 semillas aleatorias distintas.
- **Tests de Mann-Whitney U** (no paramétricos) entre todos los pares de
  clusters, para confirmar con rigor estadístico que las diferencias
  observadas entre segmentos no son producto del azar.

---

## Resultados

### KPIs del proyecto

| Métrica | Valor |
|---|---|
| Silhouette Score | 0,2012 |
| Davies-Bouldin Index | 1,6487 |
| Calinski-Harabasz Index | 822,0 |
| Estabilidad (ARI medio, 8 semillas) | 0,886 |
| Varianza explicada por PCA | 82,9% |
| Variables con diferencias significativas entre segmentos | 20 de 22 |

### Los cuatro segmentos

| Segmento | Clientes | Ingreso medio | Gasto medio | Resp. campañas | % Volumen de compra |
|---|---|---|---|---|---|
| Premium y Alta Respuesta | 565 (25,5%) | 76.428 € | 1.399 € | 51,0% | 58,8% |
| Alto Valor y Maduros | 558 (25,2%) | 58.641 € | 775 € | 28,5% | 32,2% |
| Bajo Valor y Familias Numerosas | 707 (32,0%) | 35.880 € | 104 € | 12,9% | 5,5% |
| Bajo Valor y Hogares Pequeños | 382 (17,3%) | 35.765 € | 124 € | 17,3% | 3,5% |

**Hallazgo principal:** el 49% de los clientes (los dos segmentos de menor
valor) genera solo el 9% del volumen de compra total, mientras que el
segmento Premium (25% de los clientes) aporta más de la mitad y es, además,
el más receptivo a campañas.

### Recomendación por segmento

| Segmento | Acción recomendada |
|---|---|
| Premium y Alta Respuesta | Proteger y fidelizar: programas de fidelización y cross-selling prioritario |
| Alto Valor y Maduros | Investigar y activar: testear canales/mensajes distintos ante su baja receptividad relativa |
| Bajo Valor y Familias Numerosas | Reducir inversión: menor tasa de respuesta y menor aportación al volumen de compra |
| Bajo Valor y Hogares Pequeños | Probar antes de escalar: candidato a campaña piloto de bajo coste |

---

## Entrega y puesta en producción

El pipeline completo (`scaler`, `pca`, `kmeans`, `gmm`) se serializó con
`joblib`, junto con una función `predict_segment()` que clasifica clientes
nuevos replicando exactamente las transformaciones de entrenamiento. La
metadata del modelo (semilla, métricas de validación, cluster premium,
versión de scikit-learn) se guarda en `model_metadata.json` para
trazabilidad y reproducibilidad.

Los resultados y recomendaciones se resumen en un
[informe ejecutivo](informe/informe_ejecutivo_segmentacion.pdf) orientado a
negocio, sin detalle técnico, pensado para apoyar decisiones de presupuesto
de campañas.

---

## Stack tecnológico

| Categoría | Herramientas |
|---|---|
| Lenguaje | Python 3 |
| Clustering y reducción de dimensionalidad | Scikit-learn (KMeans, PCA, GaussianMixture) |
| Validación estadística | SciPy (Mann-Whitney U), Scikit-learn (Silhouette, Davies-Bouldin, Calinski-Harabasz, ARI) |
| Interpretabilidad | RandomForest (feature importance) |
| Análisis y preprocesamiento | Pandas, NumPy |
| Visualización | Matplotlib, Seaborn |
| Serialización y producción | Joblib |

---

## Estructura del repositorio

```
clustering/
├── README.md                                    ← este archivo
├── requirements.txt
├── data/
│   └── marketing_campaign.csv                   ← dataset original (Kaggle)
├── notebook/
│   └── marketing_clustering.ipynb               ← notebook completo con todo el análisis
└── informe/
    └── informe_ejecutivo_segmentacion.pdf        ← informe ejecutivo para negocio
```

---

## Próximos pasos

El modelo actual prioriza interpretabilidad de negocio (k=4) sobre el
óptimo puramente estadístico (k=2), lo que se traduce en una estabilidad
algo menor (ARI 0,886 frente a >0,99 con k=2). El siguiente paso recomendado
es una **campaña piloto A/B** —comunicación diferenciada por segmento frente
al mensaje genérico habitual, priorizando el segmento Premium y uno de los
segmentos de bajo valor— para validar con datos reales de negocio si la
segmentación se traduce en mejor retorno de campaña, y decidir a partir de
ahí si se reasigna presupuesto de forma definitiva.
