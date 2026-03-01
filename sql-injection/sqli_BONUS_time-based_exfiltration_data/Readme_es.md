# Bonus: Exfiltración de Datos mediante Inyección SQL Basada en Tiempo

_Read this in English: [Readme.md](./Readme.md)_

[__Enlace al laboratorio__](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays)

> [!NOTE]
> **Análisis del Laboratorio:** Si buscas comprender la vulnerabilidad a fondo, justo debajo de la sección de uso encontrarás una **explicación técnica detallada (sin spoilers)** sobre el funcionamiento del ataque y la lógica de la base de datos.
>Ir directamente [allí](#metodología-y-ética)

## Script de Automatización

Este proyecto implementa un exploit de Inyección SQL Ciega (Blind SQLi) basado en el tiempo. A diferencia de las inyecciones clásicas, aquí el servidor no devuelve datos directamente. La herramienta utiliza ataques de __inferencia__, enviando payloads condicionales que inducen un retraso (`SLEEP`) en la respuesta del servidor solo si se cumple una condición específica. Al medir con precisión milimétrica la latencia de la respuesta, el script es capaz de reconstruir bases de datos completas, carácter por carácter, de forma totalmente automatizada.

### __Uso__

>Crear entorno virtual con Python (Recomendado)
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

```python
python exploit_ascii.py -h --> Muestra la ayuda

python exploit_ascii.py -t [URL] [-D] [--tables] [--sleep 3]
```

```bash
# Ejemplo avanzado: Usando un delay de 3 segundos
python exploit_ascii.py -t [URL] -D --tables --sleep 3
```

---

## Metodología y Ética

>[!important]
>__Aviso de Aprendizaje:__ A continuación se detalla el funcionamiento de la vulnerabilidad bajo un enfoque pedagógico libre de spoilers. Te animo a enfrentarte al laboratorio por tus propios medios antes de consultar este análisis. La verdadera maestría nace de la resolución persistente de problemas.

---

## Objetivo del Laboratorio

>[!note]
>Este ejercicio no forma parte del conjunto de laboratorios de Portswigger. Sin embargo, aprovechamos el laboratorio __Lab: Blind SQL injection with time delay__ para realizar un ataque que nos demuestra que es posible convertir la latencia del servidor en un oráculo binario capaz de revelar la estructura interna de una base de datos, carácter por carácter, incluso en condiciones de 'oscuridad total'.

El propósito fundamental de este proyecto es demostrar la viabilidad de la exfiltración de datos sensible en entornos donde el servidor no devuelve errores detallados ni datos directamente en la respuesta (entornos "_ciegos_" (_blind_)).

El exploit se centra en la explotación de una vulnerabilidad de Inyección SQL Ciega basada en tiempo (__Time-Based Blind SQLi__). Al no existir una salida visual de la información, el objetivo técnico es convertir al servidor en un oráculo binario que "responda" a través de la latencia.

- __Inferencia de Datos por Canal Lateral:__ Implementar una lógica de "_pregunta-respuesta_" basada en el tiempo, donde un retraso en la respuesta (`pg_sleep`) confirme la veracidad de una premisa lógica sobre los datos almacenados.

- __Exfiltración Carácter a Carácter:__ Reconstruir información estructurada (nombres de bases de datos y tablas) mediante la extracción individual de caracteres convertidos a código ASCII.

- __Optimización Algorítmica:__ Sustituir la búsqueda lineal convencional por un __algoritmo de búsqueda binaria__, minimizando el número de peticiones necesarias y reduciendo el ruido generado en los logs del servidor.

### Análisis Técnico de la Vulnerabilidad

La aplicación web es vulnerable a __Inyecciones SQL__ en el parámetro `trackingId` de las cookies. En esta ocasión no se nos devuelve nada por pantalla (__Blind SQLi__).

A diferencia de una _inyección SQL clásica_, aquí trabajamos en "__oscuridad total__". El éxito del laboratorio reside en la precisión milimétrica de la medición del tiempo de respuesta. El script debe ser capaz de distinguir entre un retraso inducido por el payload y una latencia natural de la red, garantizando que cada letra recuperada sea 100% exacta antes de pasar a la siguiente posición.

![Ejemplo de salida simple](./img/Exploit_simple.png)

#### ¿Cómo funciona?

La aplicación usa una cookie especial denominada `TrackingId` que usa para rastrear el comportamiento de los usuarios en la app.

En ocasiones normales, la aplicacioón realizaría una consulta SQL parecida a esta:
```SQL
SELECT * FROM users WHERE trackingId = '[TrackingId]'
```

Esta consulta también puede ser vulnerable si no se defiende correctamente por mucho que el usuario final no pudiera modificar de forma simple dicho parámetro.

Con herramientas como _Burp Suite_ podemos modificar el valor de dicha _cookie_. De esta forma podemos hacer algo parecido a esto:
```http
GET /filter?category=test HTTP/1.1
Host: vulnerable-website.com
Cookie: session=Hjh757UG64gd75; TrackingId=x' || pg_sleep(10) --
...
```

En la anterior consulta estamos usando el operador de concatenación de __PostgreSQL__ para unir, en la consulta original, el valor real de `TrackingId` y el _payload_ `pg_sleep(10)`.
```SQL
SELECT * FROM users WHERE trackingId = 'x' || pg_sleep(10) --'
```

La base de datos piensa: "_Debo de unir el valor de la columna `trackingId` con el resultado de la función `pg_sleep` (que será `void`)_."

Con esta consulta estamos obligando a la base de datos a realizar un time delay que nos puede servir para corroborar la existencia de la __Vulnerabilidad Ciega de Inyección SQL__, pero no podríamos exfiltrar información de forma directa ya que no se nos mostrará nada nuevo en pantalla.

Pero, ¿Y si hubiera una forma de exfiltrar información aprovechando los retardos de tiempo? Eso es lo que se conoce como __Ataques de Inyección SQL por inferencia basado en tiempo__.

Si volvemos a manipular el valor de la cookie `TrackingId` en la consulta HTTP y añadimos lo siguiente:
```http
GET /filter?category=test HTTP/1.1
Host: vulnerable-website.com
Cookie: session=Hjh757UG64gd75; TrackingId=x' || (SELECT CASE WHEN(SUBSTRING(current_database(),1,1) = 'a') THEN pg_sleep(5) ELSE pg_sleep(0) END) --
...
```

__Desglose del Payload:__
`SELECT CASE WHEN () THEN ... ELSE ... END`: Es la estructura lógica principal. Funciona como in `if/else` en programación tradicional.
- __Condición:__ ¿Es el valor del carácter en la posición $1$ igual a $a$?
- __Acción True:__ Si se cumple, ejecuta `pg_sleep(5)`, lo que genera un retraso detectable.
- __Acción False:__ Si no se cumple, ejecuta `pg_sleep(0)`, devolviendo la respuesta de inmediato.

`SUBSTRING(string, start, length)`: Esta función nos permite aislar un solo carácter de una cadena de texto (como el nombre de la base de datos).
- __Ejemplo:__ `SUBSTRING('admin',1,1)` nos devuelve 'a'.

`pg_sleep(segundos)`: Es la función encargada de pausar el proceso del motor de base de datos. Es nuestro __canal de comunicación lateral__. No necesitamos ver el dato en pantalla; el tiempo que tarda el servidor em responder nos da la respuesta "_Sí_" o "_No_".

---

## Automatización con Python (The Exploits)

Como es previsible, realizar un ataque a media o gran escala modificando manualmente el valor de la posición del carácter a extraer de la cookie en la petición HTTP no es sostenible.

Es por ello que una vez más brilla la capacidad de poder crear nuestros propios scripts para ataques avanzados como este.

Gracias a Python, podemos crear un script que realice peticiones HTTP iterando por cada carácter hasta encontrar el valor real que estabamos buscando (nombre de la base de datos, nombre de las tablas existentes en la base de datos o incluso credenciales).

### Versión Simple ([exploit.py](./exploit.py))

_Este script realiza exactamente lo explicado hasta ahora._

Realiza peticiones HTTP al servidor web modificando el valor de la __Cookie__ `TrackingId` añadiendo el _payload_, el cual se encarga de hacer las consultas SQL a la base de datos.

Se manda la misma petición HTTP a la misma posición del nombre que estemos analizando (Ej. Nombre de la base de datos actual). Hasta que la condición de comparación no sea exitosa (tarde entre 5 y 6 segundos en recibir la respuesta del servidor), no se analiza la siguiente posición del nombre.

#### Problema de la búsqueda lineal

Con la versión anterior existe un problema. Imaginemos que estamos tratando de exfiltrar el nombre de la base de datos actual (Ej. `zyx_vault`).

Hasta ahora hemos realizado una __búsqueda lineal__. Preguntamos carácter por carácter en orden ascendente: "¿Es la letra _'a'_?, ¿Es la _'b'_?, ¿Es la _'c'_? ..."

Para la priemra letra de nuestra base de datos (_'z'_), el proceso sería el siguiente:

1. __Iteración 1:__ ¿Es la _'a'_? --> Respuesta rápida (0s).
2. __Iteración 2:__ ¿Es la _'b'_? --> Respuesta rápida (0s).
3. ...
4. __Iteración 26:__ ¿Es la _'z'_? --> __Correcto!__ (el servidor tarda 5s).

__¿Cuál es el problema real?__
Aunque las primeras 25 respuestas son "rápidas", no son instantáneas. Cada petición tiene un __coste de red__ (latendia, resolución DNS, establecimiento de conexión TLS). Si cada petición tarda 100ms en ir y volver, ya hemos perdido 2.5 segundos solo en "preguntas fallidas" antes de llegar a la _'z'_.

si multiplicamos esto por un nombre de base de datos más largo, el tiempo sube exponencialmente.

---

## Método Binario

Para entender porque la __búsqueda binaria__ es tan potente en este contexto, debemos bajar a la capa de representación de datos.

>El servidor no entiende el concepto de "letra" como nosotros, sino que entiende valores numéricos.

### Estandar ASCII

Todo carácter imprimible en una base de datos PostgreSQL esta representádo por un número según el estandar __ASCII (American Standar Code for Information Interchange)__.

- El rango que nos interesa es del __32 (espacio)__ al __126 (~)__.
- Dentro de este rango se encuentran todas las letras (A-Z, a-z), números (0-9) y símbolos especiales.

Al usar la funciíon `ASCII(SUBSTRING(...))`, el script convierte la letra que queremos exfiltrar en un __entero__. Esto nos permite dejar de hacer comparaciones de igualdad (`=`) y empezar a hacer __comparaciones matemáticas de magnitud (`>` o `<`).

### Eficiencia Matemática _O(n)_ vs _O(log n)_

El _payload_ que usaremos utiliza la comparación mayor que (`>`) para interrogar al motor de búsqueda:
```SQL
... CASE WHEN (ASCII(SUBSTRING(...,1)) > 95) THEM pg_sleep(5) ...
```
Si el valor ASCII de la letra es mayor que _95_, el servidor "duerme". Si es menor o igual, responde inmediatamente.

En un ataque de tiempo, cada pregunta "fallida" tiene un coste. Si buscamos un carácter en el set ASCII imprimible (95 caracteres).

- __Búsqueda lineal (_O(n)_):__ en el peor de los casos (si la letra es `z` o `~`), el script realizaría __95 peticiones__. Si cada letra de una base de datos de 10 caracteres está al final de nuestra lista, necesitaríamos _950 peticiones_.

- __Búsqueda binaria (_O(log n)_):__ Al dividir el rango de búsqueda a la mitad en cada paso, el número máximo de peticiones para encontrar cualquier carácter es de __7__. Para la misma base de datos de 10 caracteres, solo realizaríamos _70 peticiones_. 

### Versión Binaria ([exploit_ascii.py](./exploit_ascii.py))

En este exploit aprovechamos la eficiencia matemática para realizar el ataque de la forma más limpia posible.

Imaginemos que la letra en la base de datos es la _'M'_ (valor ASCII __77__). Así es como el script infiere el resultado a bajo nivel:

1. __Estado inicial:__ Rango [32, 126]. Punto medio (`mid`): __79__.

2. __Pregunta 1:__ ¿Es el valor _ASCII > 79_?
- __Respuesta:__ _NO_ (El servidor responde en 0s).
- __Acción:__ El nuevo rango es [32, 79]. El valor está en la mitad inferior.

3. __Pregunta 2:__ Nuevo punto medio: _55_. ¿Es el _valor > 55_?
- __Respuesta:__ SÍ (El servidor tarda 5s).
- __Acción:__ El nuevo rango es [56, 79].

4. __Pregunta 3:__ Nuevo punto medio: 67. ¿Es el valor > 67?
__Respuesta:__ SÍ (El servidor tarda 5s).
__Acción:__ El nuevo rango es [68, 79].

5. __Pregunta 4:__ Nuevo punto medio: _73_. ¿Es el _valor > 73_?
__Respuesta:__ SÍ (El servidor tarda 5s).
__Acción:__ El nuevo rango es [74, 79].

6. __Pregunta 5:__ Nuevo punto medio: _76_. ¿Es el _valor > 76_?
__Respuesta:__ SÍ (El servidor tarda 5s).
__Acción:__ El nuevo rango es [77, 79].

7. __Pregunta 6:__ Nuevo punto medio: _78_. ¿Es el _valor > 78_?
__Respuesta:__ NO (El servidor responde en 0s).
__Acción:__ El rango se cierra en [77, 78].

8. __Final:__ El algoritmo determina que el valor es 77.
__Resultado:__ `chr(77)` en Python nos devuelve la __'M'__.

![Ejemplo database ascii](./img/database_ascii.png)
![Ejemplo tables ascii](./img/Tables.png)

### Ventajas

- __Reducción de la huella (Footprint):__ En el hacking ético, _menos es más_. Realizar 7 peticiones en lugar de 90 reduce drásticamente las posibilidades de ser detectado por un __WAF (Web Application Firewall)__ o un __IDS__ que busque patrones de escaneo repetitivos.

- __Independencia de Diccionario:__ Al trabajar sobre el rango numérico ASCII y no sobre una lista manual de letras, el script es capaz de exfiltrar automáticamente números, guiones, puntos y caracteres especiales sin necesidad de modificar el código.

- __Determinismo:__ El tiempo de ejecución es predecible. No dependemos de "tener suerte" y que la base de datos empiece por la letra 'A'.