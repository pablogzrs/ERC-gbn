# 🩺 ERC-gbn

### Redes bayesianas gaussianas con estructura elicitada por estudiantes de medicina: dependencias entre biomarcadores en enfermedad renal crónica

> ¿Qué pasa cuando le pides a un especialista clínico que dibuje el grafo, en lugar de aprenderlo de los datos? Este proyecto compara tres estructuras propuestas por estudiantes avanzados de medicina contra una aprendida por *hill-climbing*, y termina encontrando que el problema no es cuál estructura gana, sino que la familia de modelos no puede expresar la pregunta que a los clínicos les importa.

---

## 📋 Resumen

Una red bayesiana gaussiana (GBN) representa la distribución conjunta de un vector de variables continuas mediante un DAG, donde cada nodo es una regresión lineal sobre sus padres. Son atractivas en contexto clínico porque son interpretables: se ve *por qué* un biomarcador depende de otro, no solo qué tan bien se predice.

Aquí se aplican a nueve biomarcadores del conjunto *Chronic Kidney Disease* de UCI. La estructura no se dio por sentada: se entrevistó a cinco estudiantes avanzados de la Escuela de Medicina del campus, que produjeron tres DAGs distintos y seis consultas clínicas para evaluar la utilidad práctica del modelo.

## 🔍 Hallazgos

- **Las tres estructuras elicitadas convergieron de forma independiente** en dos decisiones: la edad como nodo raíz sin padres, y la presión arterial como nodo de mayor conectividad. Ningún algoritmo de puntuación llega solo a eso.
- **La estructura aprendida gana por BIC pero contiene arcos causalmente imposibles.** Ajuste estadístico y validez causal no coinciden, y la puntuación por sí sola no basta para elegir un grafo.
- **BIC y AIC se contradicen sobre la linealidad.** Al reemplazar cada regresión lineal por un estimador de Nadaraya–Watson, el AIC prefiere el modelo flexible y el BIC lo rechaza. Ninguno se equivoca: responden a objetivos distintos.
- **Cuatro de las seis consultas clínicas resultaron irresolubles**, todas por la misma causa: piden el diagnóstico de ERC condicionado a los biomarcadores. Una red gaussiana condicional admite variables discretas solo como padres, y el diagnóstico es, clínicamente, una consecuencia.
- **El indicador de diabetes sí aporta.** El arco `dm → bgr` mejora el BIC de forma sustantiva, lo que confirma que el marco no rechaza lo categórico; rechaza una dirección.

La conclusión de fondo: una GBN restringida a biomarcadores continuos describe bien la estructura entre ellos, pero es un instrumento inadecuado para la pregunta que los clínicos formulan en la práctica. El desajuste no está en los datos ni en la estimación, sino entre la familia de modelos y el tipo de pregunta.

## 🗂️ Estructura del repositorio

```
ERC-gbn/
├── data/
│   ├── data.csv              # Crudo de UCI (400 × 26)
│   └── data_clean.csv        # Salida del notebook (216 × 10)
├── Python/
│   └── data_cleansing.ipynb  # Depuración y exportación
├── R/
│   ├── gbn.qmd               # Análisis completo
│   └── gbn.html              # Render
├── documents/
│   ├── articulo.pdf          # Entregable
│   ├── dictionary.pdf        # Diccionario de variables
│   └── *.JPG                 # DAGs dibujados por las personas entrevistadas
├── requirements.txt
└── ERC-gbn.Rproj
```

## 🧹 Depuración

De las 26 columnas originales se conservan nueve continuas más el indicador de diabetes:

`age` · `bp` · `bgr` · `bu` · `sc` · `sod` · `pot` · `hemo` · `wc` · `dm`

1. **Redundancia hematológica.** `hemo`, `pcv` y `rc` miden aspectos muy cercanos de la misma condición, como señaló una de las personas entrevistadas. Se conserva `hemo`, que es la de menos faltantes.
2. **Registros incompletos.** Ninguna variable estaba completa y los faltantes se distribuían a lo largo de todo el conjunto, así que imputar habría equivalido a inventar perfiles clínicos. Se eliminan los incompletos.
3. **Valores fisiológicamente imposibles.** Un registro de potasio en 47 mEq/L, cuando el rango compatible con la vida no pasa de ~10; y dos lecturas de presión de 140 y 180 mmHg, incompatibles con una diastólica.

`bp` toma únicamente siete valores distintos, todos múltiplos de diez, así que difícilmente satisface el supuesto de continuidad. Se conservó de todos modos porque las personas entrevistadas le asignaron un papel central, y la desviación se discute en el artículo.

## ⚙️ Cómo reproducir

**Python** (limpieza):

```bash
pip install -r requirements.txt
jupyter lab Python/data_cleansing.ipynb
```

**R** (análisis). `graphviz.plot` necesita Rgraphviz, que vive en Bioconductor y no en CRAN:

```r
install.packages("bnlearn")
if (!requireNamespace("BiocManager", quietly = TRUE)) install.packages("BiocManager")
BiocManager::install("Rgraphviz")
```

Después, abrir `R/gbn.qmd` y renderizar. Corre en ese orden: el `.qmd` lee la salida del notebook, no el crudo.

## 🧰 Métodos

| | |
|---|---|
| Ajuste | `bn.fit` de **bnlearn**, una regresión lineal por nodo |
| Selección | BIC y AIC en convención de log-verosimilitud penalizada (**mayor es mejor**) |
| Referencia | *Hill-climbing* sobre los mismos datos, con la salvedad de que su puntuación está inflada |
| Alternativa no paramétrica | Nadaraya–Watson, ancho de banda por regla de Silverman ajustada a la dimensión, complejidad vía grados de libertad efectivos |
| Categóricas | Red gaussiana condicional (CLG), `dm` como nodo padre |
| Consultas | Muestreo lógico con `cpquery`, 10⁶ simulaciones |

## 🔭 Trabajo futuro

Un clasificador *naive Bayes* gaussiano invierte la dirección de los arcos y coloca al diagnóstico como raíz, que es justo la dirección que la familia de las CLG sí permite. Con las mismas nueve variables ya depuradas, respondería las cuatro consultas que aquí quedaron sin respuesta.

## 👥 Autores

Juan Pedro Enríquez Ruiz Velasco · Angelo Dayvis Farfán Torres · Pablo Emiliano González Rios · Eduardo Medina Meza

Departamento de Ingeniería y Ciencias, Instituto Tecnológico y de Estudios Superiores de Monterrey, Campus Guadalajara.

## 📚 Referencias

- Rubini, L. J., Soundarapandian, P., & Eswaran, P. (2015). *Chronic Kidney Disease* [Conjunto de datos]. [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/336/chronic+kidney+disease)
- Scutari, M. (2010). Learning Bayesian networks with the **bnlearn** R package. *Journal of Statistical Software*, 35(3), 1–22.
- Lauritzen, S. L., & Wermuth, N. (1989). Graphical models for associations between variables, some of which are qualitative and some quantitative. *The Annals of Statistics*, 17(1), 31–57.
- Friedman, N., Geiger, D., & Goldszmidt, M. (1997). Bayesian network classifiers. *Machine Learning*, 29(2–3), 131–163.

La bibliografía completa está en `documents/articulo.pdf`.