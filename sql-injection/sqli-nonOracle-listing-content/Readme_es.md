# Lab: Vulnerabilidad "Inyección SQL" usando ataques UNION para devolver información confidencial

_Read this in English: [Readme.md](./Readme.md)_

[__Enlace al laboratorio__](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)

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

El objetivo es realizar un ataque de __Inyección SQL con UNION__ para exfiltrar las credenciales de la tabla de usuarios, cuyos nombres (tanto de la tabla como de las columnas) son generados dinámicamente y cambian en cada instancia del laboratorio.

Para resolverlo, el exploit debe:

1. __Enumerar__ el número de columnas y sus tipos de datos.

2. __Consultar el Schema (`information_schema`)__ para descubrir los nombres aleatorios de la tabla de usuarios y sus columnas.

3. __Extraer__ todos los nombres de usuario y contraseñas.

4. __Autenticarse automáticamente__ como el usuario administrator gestionando __tokens CSRF__.

---

### Análisis Técnico de la Vulnerabilidad

La aplicación web es vulnerable a __Inyecciones SQL__ en el parámetro de filtrado de categorías. Al mostrar por pantalla la respuesta de la consulta, podemos usar ataques `UNION` para exfiltrar información directamente.

1. __Descubrimiento del Esquema (Fingerprinting)__
Dado que no conocemos los nombres de las tablas, consultamos la base de datos de metadatos de la base de datos (PostgreSQL/MySQL/Microsoft SQL Server en este caso):

- __Búsqueda de Tablas:__ Para conocer el nombre de las tablas debemos de consultar la tabla __table__ de la base de datos __information\_schema__:
```SQL
UNION SELECT table_name, NULL FROM information_schema.tables
```
También podemos usar la clausula `WHERE` para filtrar por nombres ya que, el laboratorio cambia el nombre de la tabla y columnas objetivo en cada instancia lanzada, sin embargo el nombre siempre comienza con _"users\_..."_.
```SQL
UNION SELECT table_name, NULL FROM information_schema.tables WHERE table_name LIKE 'users_%'
```
El exploit no usará la clausula `WHERE` en esta consulta, sin embargo usará expresiones regulares para el mismo cometido.

- __Búsqueda de Columnas:__ Una vez obtengamos el nombre de la tabla en la que se encuentran los usuarios, de nuevo consultaremos a la base de datos __information\_schema__ (en esta ocasión a la tabla __columns__) con la finalidad de conocer el nombre de las columnas que requerimos:
```SQL
UNION SELECT table_name, column_name FROM information_schema.columns WHERE table_name = '[table_name]'
```

2. __Exfiltración de Credenciales:__
Una vez obtenidos los nombres de la tabla y columnas necesarias (ej. `username_abc123`), construimos la consulta final con la cual se mostrará en pantalla los nombres de usuario con sus respectivas contraseñas.

Si usamos el siguiente payload `' UNION SELECT usuarios_u93qu, contraseñas_879827jf FROM users_89u902u --` en la función de filtrado por categorías, la consulta SQL real quedaría algo así:
```SQL
SELECT name, description FROM products WHERE category = '' UNION SELECT usuarios_u93qu, contraseñas_879827jf FROM users_89u902u --'
```
Aunque no es obligatorio, es recomendable dejar la consulta de categoría original vacía para que así tan solo aparezca en la pantalla los datos útiles que buscamos.

---

## 🐍 Automatización con Python (The Exploit)

Este laboratorio no es complicado de realizar manualmente. Sin embargo, si que contamos con una serie de pasos que debemos de realizar para poder explotar la vulnerabilidad con éxito.

Aquí es donde comienza a brillar realmente la capacidad de saber crear nuestros propios scripts que automaticen todo el proceso, ya que si fuera necesario realizar varios de estos exploits manualmente comenzaría a ser una tarea tediosa.

Además, crear nuestros propios scripts oermite desarrollar habilidades en Scripting para Pentesting y manejo de estados HTTP.

### Lógica de ejecución del Script

El exploit está estructurado en distinas funciones para garantizar la fiabilidad:

1. `check_columns`: Utiliza la técnica de `ORDER BY` incremental para determinar el ancho de la consulta original.
2. `check_column_data_type`: Verifica qué columnas aceptan el tipo de dato `String` insertando caracteres de prueba (`'a'`).
3. `exploit`__(Multi-stage)__:
    - __Fase 1:__ Localiza la tabla de usuarios mediante __expresiones regulares (Regex)__ (`^users_.*`). Para profundizar más en la expresión ve a la sección [__Expresiones Regulares__](#expresiones-regulares)
    - __Fase 2:__ Identifica las columnas de usuario y contraseña mediante __Regex__ (`^username_.*`, `^password_.*`).
    - __Fase 3:__ Extrae los datos y los almacena en un diccionario de Python.
4. `admin_login`: Automatiza el inicio de sesión final como `administrator`, cerrando el ciclo del ataque.

# Mitigación

- __Sentencias Preparadas:__ El uso de parámetros en las consultas SQL evitaría que los metadatos de la tabla sean accesibles mediante inyección.

- __Ofuscación del Esquema:__ Aunque el script usa Regex para encontrarlos, evitar nombres predecibles en el esquema es una capa adicional de "Seguridad por Oscuridad" (aunque no es una solución definitiva).

- __Tokens CSRF Estrictos:__ Asegurar que el token CSRF esté vinculado a la sesión y tenga una vida útil corta.

# Expresiones Regulares

En este laboratorio, los nombres de las tablas y columnas son __aleatorios__, lo que impode usar nombres estáticos en el script. Para solucionar esto, el exploit utiliza el módulo `re` de Python para identificar patrones específicos dentro del esquema de la base de datos exfiltrada.

__Patrones utilizados en el script__

| Patrón | Explicación Técnica | Objetivo en el Lab |
| :---: | :--- | :--- |
| `r"^users_.*` | Busca por cualquier cadena que __comience__ (`^`) por "_users\__" seguida de cualquier carácter (`.*`). | Identificar la tabla de usuarios generada dinámicamente |
| `r"^username_.*"` | Localiza cadenas que inicien con "_username\__". | Identificar la columna que almacena los nombres de usuario. |
| `r"^password_.*"` | Localiza cadenas que inicien con "_password\__". | Identificar la columna que contiene las credenciales. |

__¿Por qué usamos Regex en lugar de búsquedas simples?__
1. __Flexibilidad:__ El flag `re.IGNORECASE` permite que el script sea robusto frente a variaciones en las mayúsculas/minúsculas del servidor.

2. __Precisión:__ Al usar el ancla de inicio `^`, evitamos falsos positivos con otras tablas que podrían contener la palabra "users" en medio de su nombre.

3. __Automatización Total:__ Permite que el exploit sea __universal__; funcionará en cualquier instancia nueva del laboratorio de PortSwigger sin necesidad de modificar el código manualmente.

4. __Eficiencia:__ El uso de Regex con `BeautifulSoup (find(string=...))` es mucho más eficiente que descargar todo el HTML y buscar con un `if "string" in r.text`, ya que __Regex__ permite una validación estructural mucho más estricta.