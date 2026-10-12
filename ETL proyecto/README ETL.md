# Proyecto ETL – Food Delivery

Trabajo final del módulo Python para Análisis de Datos (ETL), dictado por el profesor Christian Condori.

Alumno: [Tu nombre completo]

## Contenido

1. Introducción
2. Archivos del repositorio
3. Fase 1: Extracción
4. Fase 2: Limpieza y transformación
5. Fase 3: Carga a la base de datos
6. Fase 4: Análisis para el CEO
7. Buenas prácticas aplicadas
8. Cómo ejecutar el proyecto
9. Herramientas utilizadas

---

## 1. Introducción

En este proyecto me puse en el papel de un analista de datos contratado por una startup de delivery que quiere entrar al mercado de Estados Unidos. Antes de invertir, el CEO quería responder cuatro preguntas: en qué ciudades hay más restaurantes, si el mercado es caro o barato, en qué ciudad cuesta más comer y si pagar más significa recibir mejor calidad.

Para responderlas construí un proceso ETL. ETL significa **Extract** (extraer los datos), **Transform** (limpiarlos y prepararlos) y **Load** (guardarlos en una base de datos). Al final usé consultas SQL y gráficos para presentar los resultados.

---

## 2. Archivos del repositorio

El archivo `Proyecto_final.ipynb` contiene todo el código, organizado por fases y con comentarios que explican cada paso. Este `README.md` resume lo que hice y por qué.

No subí los archivos CSV porque son los datos que entregó el profesor y pesan bastante.

---

## 3. Fase 1: Extracción

### 3.1 Lectura de los archivos

Leí los datos con la librería Pandas usando `pd.read_csv()`. Tenía un archivo con los restaurantes (`restaurants.csv`) y los menús de esos restaurantes.

Al revisar la carpeta me di cuenta de que los menús no venían en un solo archivo, sino divididos en 10 partes. Los leí uno por uno y los junté en una sola tabla con `pd.concat()`.

### 3.2 Archivo duplicado

Encontré un archivo llamado `restaurant_menus_parte_6 (1).csv` que pesaba exactamente lo mismo que `restaurant_menus_parte_6.csv`. Era una copia, así que decidí no usarlo para no duplicar información.

### 3.3 Diagnóstico de los datos

Antes de limpiar, revisé cómo venían los datos. Es como hacerle un chequeo médico a la tabla antes de trabajar con ella:

Con `.shape` vi cuántas filas y columnas tenía cada tabla. Con `.dtypes` revisé el tipo de dato de cada columna. Con `.isnull().sum()` conté los valores vacíos.

Este paso me ayudó a descubrir que el precio de los platos estaba guardado como texto y no como número, lo que impedía hacer cálculos con él.

---

## 4. Fase 2: Limpieza y transformación

### 4.1 Traducción del rango de precios

La columna de precios usaba símbolos como `$` o `$$`. Creé un diccionario que relaciona cada símbolo con un texto más claro (`$` = "Económico", `$$` = "Moderadamente caro", `$$$` = "Caro", `$$$$` = "Muy caro") y lo apliqué con `.map()`.

Al revisar el resultado encontré un restaurante con 17 signos de dólar seguidos. Como es claramente un error del scraping, lo dejé como valor vacío en lugar de inventarle una categoría.

### 4.2 Separación de la dirección

La dirección venía completa en una sola columna, con el formato: calle, ciudad, estado, código postal. La separé en tres columnas nuevas (`calle`, `ciudad` y `estado_zip`) con `.str.split(",", n=2, expand=True)`.

Usé `n=2` para que solo se corte en las dos primeras comas. Sin ese límite me salían cuatro columnas en lugar de tres. Después apliqué `.str.strip()` para quitar los espacios sobrantes, porque sin eso "Seattle" y " Seattle" se contarían como ciudades distintas.

### 4.3 Filtro de calidad

Eliminé los restaurantes sin puntaje o con puntaje cero, porque no sirven para medir calidad. También pasé las categorías a minúsculas con `.str.lower()` para que "Pizza" y "PIZZA" se trataran como lo mismo.

### 4.4 Limpieza de los menús

Primero eliminé 2,543 filas repetidas con `.drop_duplicates()`.

Luego convertí el precio de texto a número en tres pasos: quité el texto " USD", cambié la coma decimal por un punto y usé `pd.to_numeric()`. Usé el parámetro `errors="coerce"` para que, si algún valor no se podía convertir, quedara vacío en lugar de detener todo el programa.

Al revisar los resultados con `.describe()` encontré platos con **precio negativo**. Por eso, además de quitar los precios en cero como pedía el enunciado, eliminé todos los precios menores o iguales a cero.

### 4.5 Reducción y unión de tablas

Para no sobrecargar la memoria, me quedé solo con los restaurantes que tenían más de 100 calificaciones. Con eso pasé de 63,469 restaurantes a 8,702.

Después uní las dos tablas con `pd.merge()` usando el ID del restaurante. Usé un INNER JOIN para quedarme solo con los restaurantes que tuvieran menú. Como ambas tablas tenían columnas con el mismo nombre (por ejemplo "name"), agregué sufijos para diferenciar el nombre del restaurante del nombre del plato.

El resultado fue una tabla final de **63,186 filas**, donde cada fila es un plato con la información de su restaurante.

---

## 5. Fase 3: Carga a la base de datos

Con SQLAlchemy creé una base de datos SQLite llamada `delivery_insights.db` usando `create_engine()`. Después guardé la tabla con `.to_sql()` bajo el nombre `master_food_data`.

Para comprobar que la carga fue correcta, conté las filas guardadas en la base de datos y las comparé con las de mi tabla en Pandas. En ambos casos fueron 63,186.

---

## 6. Fase 4: Análisis para el CEO

### 6.1 Una lección importante

Como cada fila de la tabla es un plato y no un restaurante, no podía usar un conteo normal para contar restaurantes: un restaurante con 100 platos se contaría 100 veces. Por eso usé `COUNT(DISTINCT id)`, que cuenta cada restaurante una sola vez.

### 6.2 ¿Qué ciudades tienen más restaurantes?

Milwaukee (122) y Seattle (119) son las ciudades con más restaurantes consolidados, muy por encima de Lynnwood (40), Everett (32) y Bellevue (28). Son los mercados más grandes, pero también los más competidos.

### 6.3 ¿El mercado es caro?

No. El 73.6% de los restaurantes es Económico, el 25.7% es Moderadamente caro y solo el 0.7% es Caro. Ningún restaurante "Muy caro" pasó los filtros. La idea del CEO de que todos son caros no se cumple.

### 6.4 ¿Dónde es más caro comer?

Usé una subconsulta con `WITH` para obtener automáticamente las tres ciudades principales sin escribirlas a mano. Seattle es la más cara, con $11.97 por plato en promedio, seguida de Lynnwood ($10.82) y Milwaukee ($9.53).

### 6.5 ¿Más caro significa mejor?

Antes de calcular el promedio dejé una sola fila por restaurante, para que los que tienen menús grandes no pesaran más en el resultado. Los restaurantes Económicos (4.62) y Moderadamente caros (4.64) tienen casi el mismo puntaje. Los Caros tienen 4.45, pero son solo 4 restaurantes, así que no es suficiente para sacar una conclusión firme.

Mi conclusión es que **pagar más no garantiza una mejor experiencia**.

---

## 7. Buenas prácticas aplicadas

Durante el proyecto traté de trabajar de forma ordenada y cuidadosa. Estas son las prácticas que apliqué:

**Revisar antes de limpiar.** Hice el diagnóstico de los datos antes de modificarlos. Así encontré problemas que no estaban en el enunciado, como los precios negativos y los duplicados.

**Verificar después de cada paso.** Después de cada transformación imprimí el tamaño de la tabla o una muestra de los datos con `.head()`, para confirmar que el cambio hizo lo que esperaba.

**Documentar las decisiones.** Cada vez que tomé una decisión propia (excluir el archivo duplicado, eliminar precios negativos, dejar vacío el valor con 17 signos `$`), la expliqué en una celda de texto para que cualquiera entienda por qué lo hice.

**No inventar datos.** Cuando un valor era claramente un error, preferí dejarlo vacío en lugar de adivinar qué debería ser.

**Comentar el código.** Cada bloque de código tiene comentarios que explican qué hace, y el notebook está dividido por fases con títulos.

**Usar nombres claros.** Las variables tienen nombres que dicen qué contienen, como `df_restaurantes`, `df_menus` o `df_master`, y las columnas nuevas están en español para que se entiendan fácilmente.

**Cuidar la memoria.** Filtré los restaurantes antes de unir las tablas y borré con `del` las tablas que ya no necesitaba.

**Evitar advertencias de Pandas.** Usé `.copy()` al filtrar tablas para que Pandas no lance el aviso `SettingWithCopyWarning`, que puede causar errores difíciles de detectar.

**Comprobar la carga.** No asumí que la base de datos se guardó bien: lo comprobé contando las filas.

**Escribir consultas que se adapten solas.** En la pregunta 3 no escribí los nombres de las ciudades a mano, sino que los calculé dentro de la consulta. Si los datos cambian, la consulta sigue funcionando.

**Ser honesto con los resultados.** En la pregunta 4 aclaré que la categoría "Caro" tiene muy pocos restaurantes, en lugar de presentar el resultado como algo seguro.

---

## 8. Cómo ejecutar el proyecto

Desarrollé el proyecto en **Databricks** en lugar de Google Colab. Por eso hay tres diferencias en el código: los archivos se leen desde la ruta `/Volumes/workspace/default/delivery_raw/`, uso la función `dbutils.fs.ls()` que solo existe en Databricks, y guardé la base de datos en la carpeta `/tmp/`, porque en otras carpetas SQLite daba errores al escribir.

Para ejecutarlo en Google Colab hay que hacer tres cambios: reemplazar la variable `ruta` por la carpeta donde estén los CSV, borrar la celda que usa `dbutils` y usar `sqlite:///delivery_insights.db` como dirección de la base de datos. Después, basta con ejecutar todas las celdas en orden de arriba hacia abajo.

---

## 9. Herramientas utilizadas

Python, Pandas, SQLAlchemy, SQLite, Matplotlib, Seaborn y Databricks.
