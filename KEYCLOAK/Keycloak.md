## Documentacion

En la presente guia se desarrolla la integracion del 2FA email Authenticator

> Para la integración se hace uso de los plugin para la integracion de la autenticacion via email

El plugin usado está en el siguiente repositorio de github:

https://github.com/mesutpiskin/keycloak-2fa-email-authenticator

Se levantará de forma local y en contendor docker los elementos necesarios para probar la aplicacion. Segun la comunidad se tiene el Dockerfile que contruye ua imagen de keycloak y agrega el plugin neceasrio para la integracion con la autenticacion por email

> Para esta guia se usara, no el servidor de correo de desarrollo, sino una cuenta de correo gmail


![alt text] (authentication.png)

Duplicar el ...


Add (+) 
Add Execution
Email OTP -> Add

Action
Bind flow

En realm settings:

Email:
From -> Correo que envia los correos (*)

En realm connection:

Host -> smtp.gmail.com  | Aqui se indica el host del correo electronico
Port -> 465 | Puerto requerido del proveedor de correo
Encryption -> Enable SSL (true)
Authentication -> Enabled (true)
Username -> (*) Correo indicado como remitente
Authentication type -> Password (de requerirse)
Password -> Indicar la clave/contraseña como de aplicación
Allow UTF-8 -> Enabled (true)
