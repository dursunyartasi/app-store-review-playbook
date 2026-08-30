---
name: mobile-app-shipping-es
description: Guía completa para crear y publicar una app móvil en la App Store y Google Play, escrita a partir de rechazos reales y fallos de compilación reales. Úsala cuando el usuario quiera empezar un prototipo móvil, configurar Expo/React Native, compilar un IPA o AAB, subir a TestFlight, subir a Google Play, enviar a revisión de la App Store, redactar las notas de revisión, rellenar declaraciones de la tienda o resolver un rechazo. También se activa con - "app rechazada", "Guideline", "App Review", "TestFlight", "eas build", "altool", "perfil de aprovisionamiento", "AAB", "Play Console", "App Store Connect", "clasificación por edad", "declaración de privacidad", "capturas de pantalla".
metadata:
  version: 2.0.0
  fuente: 8 apps iOS/Android publicadas, 2026
---

# Publicar una app móvil

Todo lo que hay aquí viene de apps que llegaron a producción, y de los rechazos y fallos de compilación encontrados por el camino. Nada está parafraseado de la documentación oficial.

## Empieza averiguando dónde está la persona

Pregunta solo lo que aún necesitas. Si el mensaje ya responde a algo, sáltatelo. **Nunca hagas más de cinco preguntas.**

1. **¿Qué quieres hacer ahora mismo?**
   `prototipo nuevo` · `compilar y ponerlo en mi teléfono` · `TestFlight` · `Google Play` · `enviar a revisión` · `me han rechazado`
2. **¿Qué plataformas?** iOS, Android o ambas.
3. **¿Los usuarios inician sesión o crean contenido?** (cuentas, publicaciones, comentarios, mensajes, fotos)
4. **¿Tienes las cuentas de desarrollador?** Apple Developer cuesta 99 $/año y es imprescindible antes de que nada llegue a un dispositivo real. Google Play son 25 $ una sola vez.
5. **¿Hay backend, o hace falta uno?**

Después ve al archivo correspondiente. Lee solo ese.

| Respuesta | Lee |
|---|---|
| prototipo nuevo | `references/es/01-prototipo.md` |
| compilar / TestFlight | `references/es/02-testflight-ios.md` |
| Google Play | `references/es/03-google-play.md` |
| enviar a revisión | `references/es/04-envio-app-store.md` |
| me han rechazado | `references/es/05-rechazos.md` |
| backend, base de datos, correo | `references/es/06-infraestructura.md` |

## Qué cambian las respuestas

**La pregunta 3 es la que más pesa.** Si los usuarios pueden iniciar sesión, le debes a Apple el borrado de cuenta dentro de la app (5.1.1(v)) o te rechazarán. Si los usuarios pueden publicar algo visible para otros, le debes las cuatro obligaciones de la 1.2: denuncia visible, bloqueo, filtrado de contenido y consentimiento en mensajes directos. Son cuatro cosas distintas, y un gesto de pulsación larga no cuenta como visible. Constrúyelas en el prototipo. Añadirlas después de un rechazo cuesta un ciclo de revisión completo, es decir, días.

**La pregunta 4 condiciona todo lo demás.** Sin una cuenta de Apple de pago no hay TestFlight, ni instalación en dispositivo más allá de un perfil gratuito de 7 días, ni envío. Dilo antes de que la persona invierta un día entero compilando.

**La pregunta 5 tiene una respuesta barata.** `references/es/06-infraestructura.md` describe una configuración autoalojada que evita el pago por servicio: Coolify en un VPS normal, PostgreSQL y el plan gratuito de Brevo para el correo transaccional.

## Reglas que aplican en todas las etapas

- **Nunca escribas la contraseña ni el código 2FA de Apple o Google de la otra persona.** App Store Connect exige su propio inicio de sesión y la sesión del portal de desarrollador no se traslada. Pídele que entre, espera su confirmación y luego lleva tú los pasos de API y consola.
- **No marques casillas de declaración o consentimiento por ella.** Son afirmaciones legales sobre su app.
- **"Upload succeeded" no significa "aceptado".** Apple también rechaza durante el procesado. Consulta hasta que la compilación esté en `VALID`.
- **Prueba lo que verá quien revisa, no lo que ves tú.** La mayoría de los rechazos de `05-rechazos.md` eran cosas que funcionaban en el dispositivo y la cuenta de quien desarrollaba.
- **Antes de culpar a quien revisa, comprueba que aquello que no pudo alcanzar es alcanzable desde fuera de tu máquina.** Varios rechazos por directriz eran en realidad un 404, un feature flag apagado o un registro DNS que faltaba.

## Orden que no se puede alterar

Algunos pasos no admiten reordenación, y equivocarse cuesta compilaciones enteras:

1. Sube la **cadena de versión** antes de compilar si la actual ya está aprobada o publicada: el tren de la versión está cerrado y la subida será rechazada.
2. Comprueba que el **número de build está libre** antes de compilar, no después.
3. Cancela cualquier **envío en revisión** antes de adjuntar una compilación nueva. No puede haber dos versiones en revisión a la vez.
4. **Verifica la compilación adjunta** antes de enviar. Tras una cancelación, la llamada de adjuntar puede fallar mientras el resto del flujo continúa y envía el binario antiguo.
