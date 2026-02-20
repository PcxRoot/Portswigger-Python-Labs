# Lab: Vulnerabilidad "Inyección SQL" en la cláusula WHERE mostrando información oculta

_Read this in English: [Readme.md](./Readme.md)_

[__Enlace al laboratorio__](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)

> [!NOTE]
> **Análisis del Laboratorio:** Si buscas comprender la vulnerabilidad a fondo, justo debajo de la sección de uso encontrarás una **explicación técnica detallada (sin spoilers)** sobre el funcionamiento del ataque y la lógica de la base de datos.
>Ir directamente [allí](#metodología-y-ética)

# 🛠️ Script de Automatización

Este directorio contiene un exploit desarrollado en Python diseñado para automatizar la detección y explotación de la vulnerabilidad de este laboratorio.

### __Uso__

>Crear entonro virtual con Python (Recomendado)
```
python -m venv venv
```

>Activar el entorno virtual
>- Linux
>```bash
>source venv/bin/activate
>```
>- Windows
>```
>venv\Scripts\activate --> Símbolo del sistema (CMD)
>venv\Scripts\activate.ps1 --> PowerShell
>```


>Instalar dependencias
```
pip install -r requirements.txt
```

>Ejecutar el script
```python
python exploit.py -h --> Muestra la ayuda

python exploit -t [URL]
```

![Ejemplo de salida del exploit](./img/exploit_output.png)

---

## Metodología y Ética

>[!important]
>__Aviso de Aprendizaje:__ A continuación se detalla el funcionamiento de la vulnerabilidad bajo un enfoque pedagógico libre de spoilers. Te animo a enfrentarte al laboratorio por tus propios medios antes de consultar este análisis. La verdadera maestría nace de la resolución persistente de problemas.

---

## Objetivo del Laboratorio

El reto consiste en identificar y explotar una vulnerabilidad de Inyección SQL (SQLi) en el filtro de categorías de una aplicación web. El éxito se define al manipular la consulta lógica para forzar la visualización de productos ocultos (no lanzados), los cuales están protegidos por una restricción en la base de datos.

### Análisis Técnico de la Vulnerabilidad

La aplicación filtra los productos basándose en una categoría específica mediante una consulta dinámica. En un escenario legítimo, la base de datos ejecuta:
```SQL
SELECT * FROM products WHERE category = '[Categoría]' AND released = 1
```

__El vector de ataque: Manipulación de la lógica Booleana__
Al no existir una sanitización adecuada en el parámetro `category`, podemos inyectar operadores lógicos para alterar el flujo de la consulta.

__Payload utilizado: `' OR 1=1 --`__
Al integrar este payload, la consulta resultante en el servidor es:
```SQL
SELECT * FROM products WHERE category = '[Categoría]' OR 1=1 --' AND released = 1
```

__Desglose del exploit:__
- `' OR 1=1`: Introducimos una condición tautológica (siempre verdadera). En lógica Booleana, `FALSE OR CIERTO` resulta siempre en `CIERTO`.
- `--`: Operador de comentario en SQL (específicamente para bases de datos como PostgreSQL o MySQL). Esto anula el resto de la sentencia original, descartando la validación `AND released = 1`.

__Visualización del Impacto en los datos__
Consideremos el siguiente extracto de la tabla `products`:

| id| name| category| stock| released |

| :--- | :--- | :--- | :--- | :--- |

| 1 | Cafetera | Cocina | 238 | 1 |

| 2 | Gift Card | Gifts | 1033 | 1 |

| 3 | Cortacesped | Jardín | 122 | 0 |

__Resultado de la inyección:__
El motor de base de datos evalúa la condición para cada fila. Dado que `1=1` es una constante verdadera y la restricción `released = 1` ha sido comentada, la base de datos devuelve todas las filas de la tabla, incluyendo el producto con id: 3, exponiendo información sensible/oculta.

## Análisis del protocolo: HTTP GET Method

La vulnerabilidad se manifiesta a través del método HTTP GET. Los parámetros de entrada se transmiten directamente en la cadena de consulta (Query String) de la URL:

```http
GET /filter?category=Gifts' OR 1=1-- HTTP/1.1
```

Esta exposición facilita la manipulación directa desde el navegador o mediante scripts, ya que no requiere el envío de cuerpos de datos complejos (como en POST o JSON).

## 🐍 Automatización con Python (The Exploit)

Aunque la explotación manual es sencilla, la automatización permite desarrollar habilidades en Scripting para Pentesting y manejo de estados HTTP.

__Lógica de Ejecución del Script:__
1. __Establecimiento de Baseline:__ El script realiza una petición inicial para contar los productos visibles bajo condiciones normales (donde released = 1).
2. __Inyección Dinámica:__ Se envía una segunda petición con el payload en la URL.
3. __Verificación Basada en Contenido:__ Utilizando la librería `BeautifulSoup`, el script compara el volumen de datos recibidos. Si el número de elementos en el DOM (objetos `div` de productos) es superior al baseline, se confirma la explotación exitosa.

## Mitigación

La causa raíz de esta vulnerabilidad es la conciliación directa de datos proporcionados por el usuario en la consulta SQL. Para prevenir ataques de Inyección SQL, se deben seguir estas mejores prácticas:

### 1. Consultas Parametrizadas (Prepared Statements)

Esta es la defensa más efectiva. En lugar de concatenar cadenas, se utilizan "marcadores de posición" (`?` o `:name`). El motor de la base de datos trata el input del usuario estrictamente como __datos__, no como código ejecutable.

__Ejemplo de código seguro (en Java/PHP):__
```SQL
SELECT * FROM products WHERE category = ? AND released = 1
```

### 2. Uso de ORMs (Object-Relational Mapping)

Utilizar frameworks modernos (como Django ORM, Hibernate o Entity Framework) que manejan la abstracción de la base de datos de forma segura por defecto, aplicando parametrización automáticamente.

### 3. Validación de Entradas (Allow-listing)

Dado que las categorías suelen ser un conjunto fijo de valores (Gifts, Pets, etc.), el servidor debería validar que el parámetro category coincida exactamente con una de las opciones permitidas antes de procesar la consulta.