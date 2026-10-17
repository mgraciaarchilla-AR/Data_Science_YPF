# Data_Science_YPF
Curso de Data Science Fundación YPF

# TP_DS_10 – Patrones de demanda mensual de agentes del MEM (CAMMESA)

Proyecto grupal del curso de Data Science (Fundación YPF – Jump Educación).
Análisis de la demanda eléctrica mensual de los agentes del Mercado Eléctrico Mayorista (MEM) de Argentina, con un modelo de clustering (K-Means) y un modelo supervisado complementario.

---

## 1. Pregunta inicial

¿Existen grupos de agentes del MEM con patrones de demanda mensual similares (en nivel, estacionalidad y variabilidad), y en qué medida esos grupos coinciden con categorías reales del sistema eléctrico (tipo de agente, región, categoría tarifaria)?

## 2. Datos

* **Fuente:** `Demanda_Mensual.xlsx` de CAMMESA, exportado a `DF - Hoja 1.csv`.
* **Tamaño:** 56.696 filas y 13 columnas; 557 agentes; 47 meses (enero 2022 a noviembre 2025). No hay valores nulos.
* **Limpieza:** la demanda venía como texto con formato local (`1.537,96`) y con guiones en los meses sin demanda; se convirtió a número y los guiones se tomaron como 0.
* **Tabla intermedia:** una fila por agente y mes (suma de las filas de tarifas del mismo agente).

## 3. Metodología

1. **Ventana temporal común.** Se eligió el año calendario 2024, que es el que tiene más agentes con los 12 meses completos y demanda positiva: **474 agentes**.
2. **Variables por agente** (una fila por agente):
   - `log_demanda_media`: nivel de demanda (logaritmo de la demanda media mensual).
   - `cv`: coeficiente de variación, es decir, variabilidad mes a mes independiente del tamaño del agente.
   - `ind_estacional` = (media de verano − media de invierno) / media anual. Verano = diciembre, enero y febrero; invierno = junio, julio y agosto. Es positivo si el agente consume más en verano y negativo si consume más en invierno.
   - Se descartó `factor_carga` por su alta correlación con `cv`.
3. **Escalado.** `RobustScaler` (mediana y rango intercuartil), porque `cv` tiene una distribución muy asimétrica (asimetría 4,75).
4. **K-Means** (`n_init=10`, `random_state=42`), con evaluación de k por el método del codo y la comparación de perfiles.
5. **Visualización** con PCA de 2 componentes.
6. **Comparación** de los clusters con los tipos de agente reales.
7. **Parte supervisada:** clasificación del tipo de agente a partir de las mismas variables.

## 4. Decisiones y criterios

| Decisión | Criterio |
|---|---|
| Ventana 2024 | Mayor cantidad de agentes con 12 meses completos; el ciclo de 2024 es representativo de los demás años |
| Se conservaron los outliers | No son errores de carga sino agentes reales; 90 de los 474 agentes tienen algún valor atípico (81% de los Autogeneradores, 33% de los Distribuidores, 11% de los Grandes Usuarios). Eliminarlos distorsionaría la comparación con las categorías reales |
| Se conservaron los agentes intermitentes | 18 agentes tienen entre 1 y 11 meses con demanda; 15 son Autogeneradores (de 31). Eliminarlos implicaría perder casi la mitad de esa clase |
| `RobustScaler` | Resiste mejor los valores extremos que el escalado estándar |
| k = 4 | El codo cae fuerte de 1 a 2 clusters (−56%), menos de 2 a 3 (−26%) y luego de forma pareja (≈16% por cluster). Como el codo no define un valor único, se comparó k = 3 con k = 4. Se eligió k = 4 porque separa dos grupos de estacionalidad opuesta (pico de verano y pico de invierno) que con k = 3 quedan mezclados |
| Autogenerador = AG + AR (parte supervisada) | AR tiene solo 3 agentes; todos son empresas industriales con el mismo tipo de demanda |
| Se excluyó el tipo DM (parte supervisada) | Tiene un único agente, no se puede dividir ni validar |
| F1 macro como métrica (parte supervisada) | Las clases están desbalanceadas |

## 5. Resultados

### 5.1 Demanda agregada del sistema

La demanda total tiene dos picos por año (verano, de diciembre a febrero, e invierno, de junio a agosto) y dos valles (otoño y primavera), con una oscilación de aproximadamente ±15% y sin una tendencia clara entre 2022 y 2025. Como los picos de verano e invierno son de magnitud similar, el índice estacional del sistema completo es cercano a 0: el índice identifica a los agentes que **se apartan** de ese patrón.

### 5.2 Clusters (K-Means, k = 4, inercia 1.658,4)

| Cluster | Perfil | Agentes | % de agentes | % de la demanda |
|---|---|---|---|---|
| 0 | Demanda regular (cv medio 0,15), sin predominio de verano o invierno | 417 | 88,0% | 99,3% |
| 1 | Demanda muy irregular (cv medio 2,69) | 11 | 2,3% | 0,0% |
| 2 | Pico de invierno (índice estacional medio −0,64) | 25 | 5,3% | 0,5% |
| 3 | Pico de verano (índice estacional medio +1,06) | 21 | 4,4% | 0,2% |

Los tres clusters pequeños suman el 12% de los agentes pero solo el 0,7% de la demanda: los patrones atípicos existen, pero se concentran en agentes de bajo volumen. El PCA con 2 componentes retiene el 93,1% de la información: el primer componente (74,5%) es un eje de irregularidad (cv) y el segundo (18,6%) un eje de estacionalidad. El nivel de demanda casi no interviene, es decir, los agentes se diferencian por **cómo** consumen y no por **cuánto**.

### 5.3 Coincidencia con las categorías reales

| Cluster | Autogenerador | Distribuidor | Gran Usuario |
|---|---|---|---|
| 0 | 7 | 76 | 333 |
| 1 | 9 | 0 | 2 |
| 2 | 4 | 0 | 21 |
| 3 | 11 | 0 | 10 |

* Los 76 Distribuidores y 333 de los 366 Grandes Usuarios están en el cluster 0: el patrón mensual no permite distinguir entre ambas categorías.
* Los Autogeneradores se apartan: 24 de los 31 están en los clusters atípicos (82% del cluster 1 y 52% del cluster 3), frente al 7% que representan en la muestra completa.
* Los patrones atípicos no son exclusivos de los Autogeneradores: 33 Grandes Usuarios también están en esos clusters.
* **Conclusión:** la coincidencia con la clasificación por tipo de agente es baja; solo los Autogeneradores se diferencian de forma parcial.
* Región y categoría tarifaria: [completar si se incorpora esta comparación].

### 5.4 Estabilidad del algoritmo

Se entrenó K-Means con 10 semillas y un solo intento de inicialización. Ninguna alcanzó la inercia de la solución final (1.658,4); la más cercana (semilla 7, inercia 1.660,2) tuvo tamaños [420, 22, 21, 11], casi idénticos a los de la solución final [417, 25, 21, 11].

### 5.5 Modelo supervisado (tipo de agente)

Objetivo: clasificar a los agentes en Gran Usuario, Distribuidor y Autogenerador (473 agentes) con `log_demanda_media`, `cv` e `ind_estacional`.

| Modelo | F1 macro (validación cruzada, 5 particiones) |
|---|---|
| Base (siempre la clase mayoritaria) | 0,291 |
| Regresión logística | 0,661 |
| Árbol de decisión | 0,665 |
| Random Forest | 0,745 |

El Random Forest ajustado con `GridSearchCV` obtuvo un F1 macro de 0,753 en validación cruzada (desvío 0,086), 0,963 en entrenamiento y **0,745 en el conjunto de prueba**. La diferencia entre entrenamiento y validación indica sobreajuste, pero el resultado en prueba es prácticamente igual al de validación cruzada. La mejora respecto del Random Forest sin ajustar (0,745 a 0,753) es del orden del desvío entre particiones. La clase Distribuidor es la que más se confunde con Gran Usuario (precisión de 0,50 en prueba).

## 6. Limitaciones

* El coeficiente de variación mantiene una cola larga tras el escalado y el escalador robusto no la reduce; por eso la agrupación está muy influida por los pocos agentes de variabilidad extrema (el primer componente del PCA es casi solo `cv`). No se transformó el `cv` porque esos valores corresponden a agentes intermitentes, que son parte del patrón buscado.
* Los clusters 2 y 3 no están separados del grupo principal: forman un continuo y K-Means define fronteras sobre él. Solo el cluster 1 aparece claramente aislado.
* La partición del grupo principal depende de la inicialización de K-Means.
* Los clusters 1, 2 y 3 son pequeños (11, 25 y 21 agentes), por lo que sus porcentajes por tipo de agente son inestables.
* Para los agentes intermitentes, `cv` e `ind_estacional` se calculan sobre pocos meses.
* El análisis usa una única ventana temporal (2024).
* En la parte supervisada, el conjunto de prueba es chico (95 agentes, solo 6 Autogeneradores).