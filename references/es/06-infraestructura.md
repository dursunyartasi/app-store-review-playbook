# La infraestructura detrás de la guía

El [catálogo de rechazos](05-rechazos.md) trata de pasar la revisión. Este archivo trata de lo que corre por debajo: la infraestructura sobre la que se publican estas apps y los fallos que encontramos operándola.

Misma regla que en la guía: solo cosas que ocurrieron. Esto es **lo que usamos**, no una afirmación sobre lo que deberías usar tú.

---

## Qué ejecutamos

| Capa | Elección | Por qué |
|---|---|---|
| Móvil | **Expo / React Native** (SDK 57) | Un solo código para ambas tiendas; EAS o compilación totalmente local |
| Web / API | **Next.js** | El mismo TypeScript en los dos extremos |
| Base de datos | **PostgreSQL**, autoalojada | Coste predecible, sin sorpresas de precio por fila |
| ORM | **Prisma** | Migraciones que podemos revisar antes de que toquen producción |
| Archivos | **MinIO** (compatible con S3) o Cloudflare R2 | Objetos autoalojados, sin factura de salida de datos |
| Alojamiento | **Coolify** sobre un VPS normal | PaaS autoalojado: despliegues por git, TLS y contenedores sin precio por servicio |
| Correo | **Brevo**, plan gratuito por SMTP | 300 correos al día gratis, suficiente para OTP y notificaciones durante mucho tiempo |
| Pagos (Turquía) | **iyzico** | Tarjetas locales y pago a plazos, que Stripe no cubre allí |

Una de las apps corre sobre Supabase en lugar de PostgreSQL autoalojado. Ambas funcionan; abajo se indica qué lecciones son específicas de Supabase.

---

## Coolify y despliegue

Coolify es un PaaS autoalojado sobre tu propio VPS. Elimina la factura de alojamiento por servicio y te entrega los fallos operativos que una plataforma gestionada habría absorbido por ti.

### La presión de disco es el fallo que vas a sufrir de verdad

En cuanto el disco del servidor pasa aproximadamente del **80 %**, los despliegues fallan en la fase de exportación de capas aunque la compilación en sí haya terminado bien. Coolify lo muestra como `exit code 255` o un `DeploymentException` genérico: **la causa real queda oculta.** La exportación necesita algo así como 20 GB libres.

```bash
docker system df           # mira primero
docker builder prune -af   # la caché de compilación es lo que se puede borrar sin riesgo
```

La caché de compilación se puede limpiar sin problema (la siguiente compilación será algo más lenta). Las imágenes suelen estar referenciadas, así que podarlas libera poco. **No toques los volúmenes: son los datos de tu aplicación.** En un incidente esto llevó el disco del 92 % al 83 % y liberó 7,6 GB; el despliegue funcionó al reintentar.

Esa misma presión de disco aparece también como un `No such container: <uuid>` transitorio cuando un contenedor auxiliar muere a mitad de compilación. La presión de memoria produce el mismo síntoma, así que revisa las dos cosas.

### Otros comportamientos de despliegue que conviene conocer
- **Un despliegue recrea todos los servicios del compose**, no solo el que cambió, incluido tu contenedor de base de datos, cuyo **nombre cambia**. Todo lo que dependa de un nombre de contenedor se rompe: vuelve a resolverlo tras cada despliegue.
- **Un despliegue tarda entre 200 y 300 segundos.** Consulta hasta ver el contenedor nuevo y un HTTP 200; no des por buena la llamada que lo dispara.
- **El primer intento puede fallar sin motivo** en la fase de compose. Reintentar suele funcionar y producción no se cae.
- **Los despliegues no se disparan por webhook** de forma predeterminada: son una acción manual o de API.
- Si tu VPS está **detrás de Cloudflare**, ten en cuenta que el user agent por defecto de `urllib` está bloqueado. Usa curl o pon un user agent de navegador cuando programes contra tu propia API.

### Notas sobre Postgres
- **Supabase / PostgREST:** una tabla nueva devuelve `PGRST205 "Could not find the table in schema cache"` aunque la tabla exista. La caché REST está desactualizada. Solución: `NOTIFY pgrst, 'reload schema'`.
- **Realtime necesita `wal_level=logical`.** Con el valor `replica` por defecto, `postgres_changes` se suscribe sin problemas y luego no entrega ningún evento: un fallo silencioso que parece un bug del cliente. Cambiarlo requiere reiniciar el contenedor, así que reserva una ventana de mantenimiento.

---

## Correo en el plan gratuito, y la trampa de DNS que casi lo rompe

El plan gratuito de Brevo (300 correos al día) cubre OTP, restablecimientos de contraseña y notificaciones durante mucho tiempo. Apunta tu app a `smtp-relay.brevo.com:587`.

Para que los correos lleguen a la bandeja y no a la papelera, el dominio debe aparecer como **Authenticated** en Brevo, lo que exige:
- **DKIM** — los dos registros CNAME que da Brevo
- **DMARC** — empieza en `p=none`
- **SPF** — `include:spf.brevo.com`
- El registro TXT de verificación de Brevo

### ⚠️ La trampa del SPF
Activamos Cloudflare Email Routing para *recibir* correo en el mismo dominio. Cloudflare se ofreció a "añadir los registros que faltan", vio que ya existía un registro SPF para Brevo y propuso resolver el conflicto **borrando el registro de Brevo**.

Aceptar eso habría eliminado la autenticación de todos los correos que envía la app —OTP, notificaciones, restablecimientos de contraseña— y los habría mandado a spam. La solución es fusionar ambos includes en **un solo** registro:

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**Un dominio debe tener exactamente un registro SPF.** Más de uno incumple el RFC y rompe todo el envío. Verifícalo con `dig`, no te fíes del panel.

### La trampa del MX, y por qué es un problema de tienda
Ese mismo dominio **no tenía ningún registro MX**. Podía enviar, pero no recibir. La dirección de contacto de moderación que habíamos publicado no llegaba a nadie.

Eso no es solo un fallo de correo. La **directriz 1.2** de la App Store espera una vía funcional para denunciar contenido, y nuestras propias normas prometían responder en un plazo de tres días hábiles. Una dirección que descarta el correo en silencio es un compromiso incumplido y un riesgo en revisión. **Si publicas una dirección de contacto en tu ficha de tienda o en las normas dentro de la app, envíale un mensaje de prueba.**

Otra cosa a tener en cuenta: Brevo puede restringir el envío a una lista de IPs autorizadas. Añade tanto tu máquina de desarrollo como tu servidor, o el correo de producción morirá mientras las pruebas locales pasan.

---

## Notas de compilación móvil

Las trampas completas de compilación y subida están en el [catálogo](05-rechazos.md). Las decisiones de infraestructura que hay detrás:

- **Las compilaciones locales ganan a EAS remoto cuando estás iterando.** Las colas remotas se llenan, y EAS en modo no interactivo no actualiza credenciales, así que un perfil de aprovisionamiento anterior a una capacidad nueva te bloquea sin salida. `xcodebuild` local más `xcrun altool` es la vía de escape.
- **Piensa en `.env` desde el punto de vista de EAS.** Un `.env` ignorado nunca llega al archive, lo que produce variables vacías y un fallo al arrancar que solo aparece en compilaciones standalone.
- **Las compilaciones locales de Android necesitan `ANDROID_HOME`** o Gradle informa "SDK location not found".
- **Automatiza la subida a Play con una cuenta de servicio** (`eas.json > submit.android`). Subir el `.aab` a mano es el paso que más tiempo sigue siendo manual, y la automatización de navegador no ayuda: los archivos superan con creces cualquier límite de subida.

---

## De dónde sale el VPS

Coolify necesita un VPS normal con acceso root; no hace falta una plataforma gestionada. Sirve cualquier proveedor con Docker y una IP pública. Dimensionando a partir de lo que ejecutamos: una instancia pequeña basta para la app, pero **dale al disco más margen del que parece necesario**, porque el fallo de exportación de capas descrito arriba es un problema de disco, no de CPU. Presupuesta 20 GB de holgura por encima de tus imágenes.

Los nuestros corren en Hostinger. **Enlace de referido — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — usarlo le da una comisión al autor y a ti un descuento. No es un requisito: Coolify funciona en cualquier proveedor con Docker y acceso root, y nada en esta guía depende del alojamiento.

---

## Cómo conecta esto con la revisión

Varios de los rechazos del catálogo eran problemas de infraestructura disfrazados de número de directriz:

| Parecía | En realidad era |
|---|---|
| Directriz 1.2, sin forma de denunciar contenido | Una dirección de contacto publicada sin registro MX |
| Envío a Play rechazado | La URL declarada de política de privacidad daba 404 |
| 2.1 App Completeness, "la app peta al arrancar" | El `.env` nunca llegó a la compilación |
| 2.1, "no pudimos acceder a la función" | Un feature flag apagado en producción |

Antes de culpar a quien revisa, comprueba si aquello que no pudo alcanzar es realmente alcanzable desde fuera de tu máquina.
