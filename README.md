# Modelos de Machine Learning para Ecobici

Este repositorio contiene los notebooks utilizados para entrenar y evaluar modelos de regresión que predicen la **duración de un viaje de Ecobici en segundos**. El análisis utiliza recorridos de la Ciudad de Buenos Aires correspondientes al período 2022-2024.

## Objetivo

El problema se plantea como una regresión supervisada:

- **Variable objetivo:** `duracion_seg`.
- **Variables predictoras:** distancia, hora de inicio, día de la semana, indicador de fin de semana, mes y modelo de bicicleta.
- **Modelos evaluados:** regresión lineal como baseline y Random Forest como modelo no lineal.
- **Métricas:** R², MSE, RMSE y MAE. RMSE y MAE también se expresan en minutos para facilitar su interpretación.

## Notebooks

| Notebook | Dataset esperado | Descripción |
| --- | --- | --- |
| `ml_duracion_recorridos_BA.ipynb` | `ml_duracion_recorridos.parquet` | Entrena con el conjunto completo, que conserva viajes extremos. |
| `ml_duracion_recorridos_filtrado_mateBA.ipynb` | `ml_duracion_recorridos_filtrado_mate.parquet` | Entrena con datos filtrados; excluye duraciones extremas y es la variante recomendada para comparar los modelos actuales. |

Los archivos Parquet no están incluidos en este repositorio. Son generados previamente en la capa *gold* del flujo de datos mediante dbt y DuckDB.

## Estructura esperada de los datos

Cada Parquet debe contener las siguientes columnas:

| Columna | Uso |
| --- | --- |
| `id_recorrido` | Identificador del viaje; se excluye del entrenamiento. |
| `duracion_seg` | Duración real en segundos y variable objetivo. |
| `distancia_km` | Distancia del recorrido en kilómetros. |
| `hora_origen` | Hora de inicio, entre 0 y 23. |
| `dia_semana` | Día de la semana, entre 1 y 7. |
| `es_fin_de_semana` | Indicador binario de fin de semana. |
| `mes` | Mes, entre 1 y 12. |
| `modelo_bicicleta` | Categoría de bicicleta (`FIT` o `ICONIC`). |

Antes de entrenar, `modelo_bicicleta` se transforma mediante *one-hot encoding*. La categoría resultante observada en los notebooks es `modelo_bicicleta_ICONIC`.

## Ejecución en Google Colab

1. Subir el Parquet correspondiente a Google Drive.
2. Abrir en Colab el notebook que se desea ejecutar.
3. Ajustar la variable `RUTA_PARQUET` para que apunte a la ubicación real del archivo. Por ejemplo:

   ```python
   RUTA_PARQUET = "/content/drive/MyDrive/tpo_datos/ml_duracion_recorridos_filtrado_mate.parquet"
   ```

4. Ejecutar todas las celdas en orden. Colab solicitará autorización para montar Google Drive.

Los notebooks instalan `pyarrow` y utilizan `pandas`, `numpy`, `matplotlib` y `scikit-learn`. Para leer el dataset y entrenar con todos los registros se necesita una sesión con memoria suficiente. Si Colab se queda sin RAM, se puede habilitar la muestra incluida en el notebook:

```python
df = df.sample(500_000, random_state=42)
```

El uso de una muestra cambia las métricas finales, aunque permite validar rápidamente el flujo completo.

## Flujo de generación

La ejecución realiza las siguientes etapas:

1. Montaje de Google Drive y lectura del Parquet.
2. Análisis exploratorio de distribuciones, outliers y relación entre distancia y duración.
3. Separación entre variables predictoras (`X`) y objetivo (`y`).
4. Codificación de `modelo_bicicleta`.
5. División aleatoria reproducible: 80 % entrenamiento y 20 % prueba (`random_state=42`).
6. Entrenamiento de una regresión lineal.
7. Entrenamiento de un `RandomForestRegressor` con 50 árboles, profundidad máxima de 15 y uso de todos los núcleos disponibles.
8. Evaluación sobre el conjunto de prueba y comparación de métricas.
9. Visualización de la importancia de las variables del Random Forest.

Actualmente, los modelos quedan disponibles en memoria como `modelo_lineal` y `modelo_rf` mientras la sesión está activa. Los notebooks **no guardan un artefacto entrenado** (`.joblib`, `.pkl`, etc.); para reutilizar un modelo es necesario volver a ejecutar el entrenamiento o incorporar una etapa de serialización.

## Resultados registrados

Los siguientes valores corresponden a las salidas guardadas en los notebooks y pueden variar si cambia el dataset, su filtrado o las versiones de las dependencias.

| Dataset | Modelo | R² | RMSE | MAE |
| --- | --- | ---: | ---: | ---: |
| Completo (9.065.317 viajes) | Regresión lineal | 0,0862 | 25,63 min | 9,98 min |
| Completo (9.065.317 viajes) | Random Forest | 0,1143 | 25,23 min | 9,10 min |
| Filtrado (8.721.508 viajes) | Regresión lineal | 0,2441 | 12,96 min | 8,21 min |
| Filtrado (8.721.508 viajes) | Random Forest | 0,3737 | 11,80 min | 7,02 min |

En el dataset filtrado, el Random Forest obtiene el mejor resultado. `distancia_km` concentra aproximadamente el 84,65 % de la importancia calculada por el modelo, muy por encima del resto de las variables.

## Reproducibilidad y consideraciones

- La partición de datos y el Random Forest utilizan `random_state=42`.
- El entrenamiento completo del Random Forest tardó cerca de 18 minutos en las ejecuciones registradas con dos núcleos; el tiempo depende del entorno.
- El split actual es aleatorio, no temporal. Por lo tanto, las métricas miden generalización sobre viajes mezclados del período y no necesariamente el desempeño sobre meses futuros.
- Un RMSE mucho mayor que el MAE indica que algunos viajes extremos concentran errores grandes. Esto explica la diferencia entre los resultados del dataset completo y el filtrado.
- Si cambia el esquema de la capa *gold*, se deben revisar `df.columns`, los nombres utilizados en el notebook y las reglas de filtrado antes de reentrenar.
