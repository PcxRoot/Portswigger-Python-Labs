# Lab: Vulnerabilidad "Inyección SQL" devolviendo datos de la base de datos Oracle

_Read this in English: [Readme.md](Readme.md)_

[__Enlace al laboratorio__](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)

> [!NOTE]
> **Análisis del Laboratorio:** Si buscas comprender la vulnerabilidad a fondo, justo debajo de la sección de uso encontrarás una **explicación técnica detallada (sin spoilers)** sobre el funcionamiento del ataque y la lógica de la base de datos.
>Ir directamente [allí](#metodología-y-ética)

# Script de Automatización

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

![Ejemplo de salida del exploit](./img/Exploit_output.png)

---

## Metodología y Ética

>[!important]
>__Aviso de Aprendizaje:__ A continuación se detalla el funcionamiento de la vulnerabilidad bajo un enfoque pedagógico libre de spoilers. Te animo a enfrentarte al laboratorio por tus propios medios antes de consultar este análisis. La verdadera maestría nace de la resolución persistente de problemas.

---

## Objetivo del Laboratorio

El objetivo es explotar una vulnerabilidad de __Inyección SQL vía UNION__ para recuperar la versión de la base de datos en un entorno __Oracle__. Para lograrlo, el exploit debe:

1. Determinar el número de columnas devueltas por la consulta original.

2. Identificar qué columna es compatible con tipos de datos de texto (_String_).

3. Extraer la información de versión consultando la tabla del sistema `v$version`.

### Análisis Técnico de la Vulnerabilidad

La aplicación filtra productos por categoría. La consulta interna en Oracle se ve similar a esto:
```SQL
SELECT name, description FROM products WHERE category = '[user_input]' AND released = 1
```

__1. Determianción de Columnas (`ORDER BY`)__
Para usar un ataque `UNION`, ambas consultas deben tener el mismo número de columnas. Utilizamos la cláusula `ORDER BY X` para encontrar el límite donde la base de datos lanza un error (HTTP 500).

- `ORDER BY 1` -- `200 OK`
- `ORDER BY 2` -- `200 OK`
- `ORDER BY 3` -- `500 Internal Server Error` (La tabla tiene solo dos columnas)

__2. Verificación de Tipos de datos__
En Oracle, cada columna en un `UNION` debe ser compatible con el tipo de datos de la columna correspondiente de la consulta original. Además, Oracle __requiere__ la cláusula `FROM` siempre, por lo que usamos la tabla virtual `DUAL`.

- __Payload de prueba:__ `' UNION SELECT 'a', NULL FROM DUAL --`

__3. Extracción de Versión (Oracle Specific)__
Una vez identificada la columna que acepta texto, consultamos `v$version`, que es una vista especial en Oracle que contiene el `banner` del software.

## Análisis del protocolo: HTTP GET Method

La vulnerabilidad se manifiesta a través del método HTTP GET. Los parámetros de entrada se transmiten directamente en la cadena de consulta (Query String) de la URL:

```http
GET /filter?category=Gifts' OR 1=1-- HTTP/1.1
```

Esta exposición facilita la manipulación directa desde el navegador o mediante scripts, ya que no requiere el envío de cuerpos de datos complejos (como en POST o JSON).

## 🐍 Automatización con Python (The Exploit)

Aunque la explotación manual es sencilla, la automatización permite desarrollar habilidades en Scripting para Pentesting y manejo de estados HTTP.

El script automatiza el proceso de "fuerza bruta" lógica para encontrar la estructura de la base de datos:

1. __Fase de Enumeración (`ORDER BY` Loop)__: El script itera del 1 al 5 enviando peticiones. Al detectar un estado `HTTP 500`, calcula que el número de columnas es $n-1$.

2. __Fase de Fingerprinting de Datos:__ Crea una lista de valores `NULL`. Sustituye sistemáticamente cada posición por un carácter `'a'` y verifica si el servidor responde con un `HTTP 200 (OK)`. Esto confirma que esa columna puede mostrar texto.

3. __Inyección Final y Extracción:__ Construye el payload final usando `UNION SELECT` apuntando a `BANNER` de la tabla `v$version`.

4. __Validación de Resultados:__ Utiliza `BeautifulSoup` para buscar la cadena _"PL/SQL"_ dentro de la tabla de descripción de la página, confirmando que la versión del software ha sido volcada con éxito.

## Mitigación

La prevención contra ataques `UNION` sigue los mismos principios que otras inyecciones SQL, pero enfatiza el control del esquema:

- __Consultas Parametrizadas:__ Evitan que el atacante pueda cerrar la comilla e iniciar una nueva sentencia `UNION`.

- __Principio de Menor Privilegio:__ La cuenta de usuario de la base de datos que usa la web no debería tener permisos de lectura sobre tablas de sistema como `v$version` o `v$instance`.

- __Validación Estricta de Tipos:__ Asegurar que el input del usuario solo contenga valores esperados (alfanuméricos simples) antes de procesarlos.