# Lab: Bypass simple de 2FA

_Read this in English: [Readme.md](./Readme.md)_

[__Enlace al laboratorio__](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)

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

![Output Example](./img/Output%20exploit.png)

---

## Metodología y Ética

>[!important]
>__Aviso de Aprendizaje:__ A continuación se detalla el funcionamiento de la vulnerabilidad bajo un enfoque pedagógico libre de spoilers. Te animo a enfrentarte al laboratorio por tus propios medios antes de consultar este análisis. La verdadera maestría nace de la resolución persistente de problemas.

---

## Objetivo del Laboratorio

El objetivo del laboratorio es saltrase el doble factor de autenticación impuesto en la redirección tras el inicio de sesión por credenciales correctas.

Para resolverlo, el exploit debe:

1. __Inicio de sesión con credenciales:__ Iniciamos sesión con las credenciales que nos da el mismo laboratorio.

2. __Bypass del 2FA:__ Tras iniciar sesión con las credenciales, nos redirigiremos directamente a `/my-account`.

### Análisis técnico de la vulnerabilidad

La aplicación web verifica la identidad del usuario, tras introducir las credenciales correctas, redirigiendonos a un _endpoint_ en el cual deberemos de introducir un código único y temporal que nos enviará la aplicación web al correo electrónico.

Nosotros contamos con las credenciales de la víctima, pero no tenemos acceso al email del mismo, por lo que esto efectivamente nos impediría acceder a la cuenta.

El fallo en este escenario no reside en el código 2FA en sí, sino en una __gestión de estado de sesión deficiente__. El servidor asume que el usuario solo llegará a su panel de control si sigue el flujo de navegación previsto, pero no valida este estado de forma estricta en el lado del servidor.

#### El Flujo de Autenticación Roto

En una implementación segura, el acceso a /my-account debería estar restringido por un "_flag_" o estado que confirme que ambos factores de autenticación han sido completados. Sin embargo, en este laboratorio ocurre lo siguiente:

1. __Paso 1 (Credenciales):__ Introducimos usuario y contraseña válidos. El servidor nos identifica y genera una sesión activa.

2. __Paso 2 (Redirección):__ El servidor nos redirige automáticamente a `/login2` para solicitar el código 2FA.

3. __El Fallo de Lógica:__ Aunque estamos en la pantalla del 2FA, la sesión ya se considera "autenticada" para el _endpoint_ `/my-account`. El servidor confía en la cookie de sesión tras el primer paso y no verifica si el segundo factor ha sido validado antes de servir contenido protegido.

## 🐍 Automatización con Python (The Exploit)

Aunque la explotación manual es sencilla, la automatización permite desarrollar habilidades en Scripting para Pentesting y manejo de estados HTTP.

__Lógica de Ejecución del Script:__
1. __Normalización de URL:__ Admite que la URL que le pasemos como parámetro termine con o sin "`/`".

2. __Petición `POST` con credenciales:__ Realiza la primera petición `POST` con las credenciales correctas.

3. __Bypass de 2FA:__ Una vez introducidas las credenciales y redirigidos al _endpoint_ `/login2`, realizamos una petición `GET` al _endpoint_ `/my-account`. Como para el servidor ya estamos autenticados realmente desde que introducimos las cerdenciales y usa el 2FA como un método extra que no debe de saltarse el usuario, nor permite el acceso.

4. __Verificación de éxito:__ Para corroborar que el _exploit_ ha sido satisfactorio, busca en la respuesta del servidor el nombre de usuario "_carlos_". Si existe en el primer párrafo de la respuesta, el exploit ha sido exitoso, de lo contrario falló.

## Mitigación

Para corregir esto, el servidor debería implementar un __estado de autenticación parcial__. La sesión no debería dar acceso a recursos protegidos hasta que un atributo específico (ej. `mfa_verified: true`) sea actualizado en la base de datos de sesiones tras introducir el código correcto.