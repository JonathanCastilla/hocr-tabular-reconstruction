# Pipeline de Reconstrucción Tabular HOCR a CSV mediante Agrupamiento Espacial e IA Generativa

- **Autor:** Jonathan Eduardo Castilla Zamora 🙋
- **Objetivo:** Extracción avanzada de datos financieros a partir de documentos desestructurados.

---

## 🚀 Ejecución Interactiva en la Nube

La arquitectura completa ha sido empaquetada en un entorno interactivo. Puede auditar el código, visualizar los *dataframes* y replicar el *pipeline* sin necesidad de configuraciones locales haciendo clic a continuación:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1EsVasqdk1IcQuMZHr-kar1xvDivFCEjU?usp=sharing)

## 1. Resumen
Este proyecto implementa una solución automatizada, resiliente y escalable para la extracción, estructuración y refinamiento de datos tabulares a partir de archivos en formato HOCR. El desafío principal radica en inferir la semántica tabular (filas y columnas) a partir de coordenadas puramente espaciales de *bounding boxes* generadas por motores OCR.

Para superar la fragilidad de las heurísticas basadas en umbrales de píxeles estáticos (*hardcoding*), esta solución propone una arquitectura híbrida de dos etapas: **Machine Learning No Supervisado** para la reconstrucción topológica y **Modelos Fundacionales (LLMs)** para la transducción semántica y estandarización de datos financieros.

---

## 2. Arquitectura del Sistema y Metodología

El flujo de procesamiento (*pipeline*) ha sido desacoplado en tres módulos funcionales independientes siguiendo el principio de *Separation of Concerns*:

### 2.1. Módulo de Extracción Espacial (`extract_spatial_tokens`)
El formato HOCR anida la información posicional dentro de los atributos de las etiquetas HTML. 
* **Análisis de Árbol DOM:** Se utiliza `BeautifulSoup4` para parsear de manera robusta el documento frente a HTML mal formado.
* **Normalización Geométrica:** En lugar de emplear coordenadas absolutas superiores o inferiores, el algoritmo calcula el centroide vertical (`y_centroid`) de cada *token*. Esto mitiga la varianza geométrica introducida por palabras con diferentes alturas de fuente (ej. letras mayúsculas frente a minúsculas), garantizando una estricta alineación horizontal.

### 2.2. Módulo de Reconstrucción Topológica (`cluster_grid_topology`)
Una vez extraídos los nodos en un espacio continuo bidimensional, se discretizan en una cuadrícula tabular lógica.
* **Agrupamiento de Densidad (DBSCAN):** Se aplica el algoritmo espacial DBSCAN (`scikit-learn`) de manera independiente sobre los ejes X e Y. Este enfoque evalúa la densidad espacial para inferir dinámicamente las filas y columnas, absorbiendo como "ruido" las desviaciones causadas por inclinaciones (*skew*) naturales del documento.
* **Intersección Matricial:** Las celdas se ensamblan mapeando la intersección matemática entre los identificadores de clústeres de filas y columnas resultantes.

### 2.3. Módulo de Refinamiento Semántico (`refine_semantics_with_llm`)
La matriz espacial pura presenta deficiencias típicas del reconocimiento óptico a nivel de carácter y carece de formato estandarizado. Se integra el modelo `gemini-2.5-flash` a través de la API de Google Generative AI actuando como un transductor determinista. 

Mediante *Prompt Engineering* avanzado (Zero-shot con restricciones estrictas), el modelo aplica reglas de sanitización de datos:
* **Preservación de Asimetría:** Mantiene intactos títulos y categorías generales que abarcan múltiples columnas en el documento original.
* **Escape de Texto (*Text Wrapping*):** Encapsula cadenas que contienen comas gramaticales (ej. razones sociales o direcciones) entre comillas dobles para evitar la fragmentación de columnas y la corrupción del estándar CSV.
* **Estandarización Financiera:** Elimina comas que actúan como separadores numéricos de miles, unifica cifras fragmentadas por el subrayado del OCR e inyecta símbolos monetarios de manera consistente.
* **Alineación Vertical Estricta:** Implementa relleno (*padding*) con celdas vacías (`,,`) para asegurar que los montos financieros no sufran desplazamientos espaciales y se mantengan exactamente bajo sus encabezados correspondientes.

---

## 3. Consideraciones de Ingeniería y MLOps

* **Resiliencia Computacional (*Graceful Degradation*):** El sistema implementa bloques lógicos de contención (`try-except`). Ante fallos de latencia, problemas de red o cuota en la API del LLM, el sistema no colapsa; degrada suavemente retornando la topología espacial cruda generada por DBSCAN.
* **Codificación Anti-Mojibake (BOM para Excel):** Para asegurar la correcta interpretación de caracteres del idioma español en *software* de usuario final, el orquestador guarda los archivos procesados bajo la codificación `utf-8-sig`. Esta técnica inyecta un *Byte Order Mark* (BOM) invisible, solucionando la corrupción de caracteres a nivel de sistema de archivos sin depender de la IA.
* **Procesamiento en Lote (*Batch Orchestration*):** El controlador principal escanea el directorio objetivo, orquesta la ejecución secuencial de los tres módulos sobre cada documento, provee auditoría visual en tiempo de ejecución (`pandas`) y dispara automáticamente las descargas.

---

## 4. Requisitos del Entorno

Las versiones exactas utilizadas durante el desarrollo y auditoría del entorno son las siguientes:

```text
Python:               3.12.12
Pandas:               2.2.2
BeautifulSoup4:       4.13.5
Scikit-learn (DBSCAN):1.6.1
Google Generative AI: 0.8.6
```

## 5. Instrucciones de Ejecución

El código ha sido empaquetado y adaptado para su revisión interactiva de tipo *Plug & Play* mediante **Google Colab**.

1. Abra el *Notebook* proporcionado en la entrega a través del botón *Open in Colab*.
2. Ejecute la celda de inicialización del entorno. El *script* clonará automáticamente este repositorio desde GitHub para auto-abastecerse de los archivos de prueba (`.hocr`), sin requerir permisos ni montaje de su Google Drive personal.
3. El sistema solicitará de manera segura (vía `getpass`) su API Key de **Google Gemini** para habilitar el transductor semántico. Ingrese su clave cuando el entorno lo indique.
4. Ejecute el orquestador maestro; los archivos `.csv` finales se procesarán en la memoria temporal de la máquina virtual y se forzará su descarga automática al equipo local del evaluador.