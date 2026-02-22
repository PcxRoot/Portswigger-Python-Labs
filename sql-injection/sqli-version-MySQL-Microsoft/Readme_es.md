# Lab: Vulnerabilidad "Inyección SQL" devolviendo datos de la base de datos MySQL o Microsoft

_Read this in English: [Readme.md](Readme.md)_

[__Enlace al laboratorio__](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)

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

![Ejemplo de salida del exploit](./img/exploit_output.png)

---

## Metodología y Ética

>[!important]
>__Aviso de Aprendizaje:__ A continuación se detalla el funcionamiento de la vulnerabilidad bajo un enfoque pedagógico libre de spoilers. Te animo a enfrentarte al laboratorio por tus propios medios antes de consultar este análisis. La verdadera maestría nace de la resolución persistente de problemas.

---

## Objetivo del Laboratorio

El reto consiste en explotar una vulnerabilidad de __Inyección SQL (UNION-Based)__ para extraer la versión del software de la base de datos. A diferencia de _Oracle_, este entorno utiliza sintaxis compatible con __MySQL/Microsoft SQL Server__, lo que requiere:

1. Determinar el número de columnas mediante `ORDER BY`.

2. Identificar columnas que acepten tipos de datos `String`.

3. Utilizar la variable global `@@version` para obtener la información del sistema.

### Análisis Técnico de la Vulnerabilidad

La aplicación procesa el parámetro `category` de forma insegura, permitiendo la inyección de comandos SQL que alteran el conjunto de resultados original.

1. __Enumeración de Columnas__
Utilizamos `ORDER BY X #` para forzar un error en el servidor cuando el índice supera el número real de columnas. El carácter `#` (codificado como `%23`) es esencial en __MySQL__ para comentar el resto de la consulta original.

2. __Fingerprinting de Tipos de Datos__
Para que un `UNION SELECT` funcione, las columnas inyectadas deben coincidir en tipo con las originales. El script prueba sistemáticamente insertando `'a'` en cada posición `NULL`. Un código de estado `HTTP 200` indica que la columna es apta para mostrar texto.

3. __Exfiltración del Sistema__
Una vez identificada la columna inyectable, se solicita la versión del servidor. En este entorno, la constante `@@version` devuelve detalles específicos del sistema operativo y la versión del motor (ej. detalles de __Ubuntu/MySQL__).

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

3. __Inyección Final y Extracción:__ Construye el payload final usando `UNION SELECT` apuntando a `@@version`. A diferencia de _Oracle_, MySQL y Microsoft SQL Server no requieren de usar la clausula `FROM` en cada consulta `SELECT`. Por lo que el payload final se verá algo como:
```HTTP
GET /filter?category=test' UNION SELECT @@version, NULL #
```

4. __Validación de Resultados:__ Utiliza `BeautifulSoup` para buscar la cadena _"0ubuntu0"_ dentro de la tabla de descripción de la página, confirmando que la versión del software ha sido volcada con éxito.

## Mitigación

La prevención contra ataques `UNION` sigue los mismos principios que otras inyecciones SQL, pero enfatiza el control del esquema:

- __Consultas Parametrizadas:__ La única defensa definitiva contra SQLi.

- __Input Validation:__ Implementar filtros que rechacen caracteres especiales como `'`, `#` o palabras clave de SQL.

- __WAF (Web Application Firewall):__ Configurar reglas que detecten patrones de ataques `UNION` y `ORDER BY`.