# De cero a un prototipo funcionando

Objetivo: una app corriendo en el teléfono de la persona, construida de forma que los requisitos de tienda de `05-rechazos.md` ya estén cumplidos en lugar de añadidos a posteriori.

## Decisiones previas al primer archivo

**Flujo managed o bare.** Managed mantiene `ios/` y `android/` como carpetas generadas; bare las incorpora al repositorio. Bare te da control nativo y te quita esto: **`app.json` deja de ser la fuente de verdad.** Las cadenas de propósito de los permisos ya no se propagan a `Info.plist`, y la versión pasa a venir de `CFBundleShortVersionString` en `Info.plist` más `MARKETING_VERSION` en el pbxproj. Ambas cosas nos han costado rechazos. Empieza en managed salvo que un módulo nativo te obligue.

**Identificador de bundle.** Elígelo ahora y con seguridad: en cuanto existe un registro en App Store Connect, **el bundle ID es permanente.** Usa DNS inverso sobre un dominio que controles.

**El nombre que mostrará Apple.** En una cuenta individual de Apple Developer, el nombre de desarrollador que aparece en la App Store es tu nombre legal tal como quedó registrado. Los caracteres no ASCII pueden perderse silenciosamente al registrarse (a nosotros se nos cayeron los diacríticos turcos), y el arreglo autoservicio de App Store Connect **no funciona**: te arrastra a la verificación de dirección y a la cadena del Paid Apps Agreement, y nunca guarda el nombre. Corregirlo exige una solicitud de soporte con verificación de identidad. **Revisa la ortografía carácter a carácter durante el registro.**

## Andamiaje

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # escanea el QR con Expo Go
```

Expo Go basta hasta que añadas un módulo nativo o necesites una compilación firmada. A partir de ahí necesitas una development build o un archive real.

## Integra ya los requisitos de tienda

Son baratos el primer día y caros después de un rechazo.

**Si los usuarios pueden iniciar sesión — borrado de cuenta (5.1.1(v)).** Debe ser accesible dentro de la app, inmediato y permanente. Sin desactivación, sin periodo de espera, sin "escribe a soporte para borrarla", sin redirección a una web. Pide la contraseña de nuevo, muestra una confirmación destructiva, enumera lo que se va a borrar y revoca también los permisos de terceros en el lado del proveedor.

**Si los usuarios pueden publicar algo — los cuatro requisitos de la 1.2.** Un control visible en cada mensaje, publicación y comentario (un botón "⋯"; el gesto de pulsación larga es invisible para quien revisa y fue rechazado), bloqueo que funcione en ambos sentidos, filtrado de contenido en **todos** los endpoints de escritura, y un paso de consentimiento antes de que un desconocido pueda enviar más de un mensaje directo.

**Los enlaces legales deben ser pulsables (2.1(a)).** "Al registrarte aceptas los Términos" como texto plano es un rechazo. Que sean enlaces reales, que se abran en un navegador dentro de la app en lugar de expulsar a Safari, y que estén también en la pantalla de inicio de sesión, no solo en la de registro.

**Permisos.** No pidas nada que no uses. Solicitar acceso completo a la fototeca antes de abrir el selector fue un rechazo: el selector moderno de iOS devuelve una foto sin ningún permiso. Las pantallas de contexto previo no deben usar textos dirigidos: "Continuar", no "Usar mi ubicación".

**Una dirección de contacto que reciba correo de verdad.** Si publicas una en la ficha o en las normas dentro de la app, el dominio necesita un registro MX. Mira la trampa del MX en `06-infraestructura.md`: el nuestro podía enviar pero no recibir, así que la dirección de moderación de nuestras normas publicadas no llegaba a nadie.

## Variables de entorno

```
.env            → en .gitignore, como siempre
.easignore      → esto es lo que lee EAS, y sustituye a .gitignore
```

**Un `.env` ignorado nunca llega al archive de EAS.** El bundle sale con variables vacías y la app peta al arrancar, **solo** en compilaciones standalone, así que el simulador y el dev client se ven perfectos. En una de nuestras apps petaban absolutamente todas las builds por esto antes de que lo encontráramos. O configuras variables de entorno en EAS, o te aseguras de que `.easignore` no excluye lo que la compilación necesita.

## Llévalo a un dispositivo real

Un simulador no demuestra que la app funcione. Los fallos que solo ocurren en standalone son justo los que llegan a quien revisa:

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # detecta errores de import pronto
```

Después compila e instala por cable, y mira el log con `devicectl --console`. Un módulo nativo importado dinámicamente con `import()` que no está instalado es invisible en desarrollo —Metro lo sirve— y en standalone peta con `RCTFatalException: Cannot find module`.

## Antes de seguir

Ejecuta `npx tsc --noEmit` y tus tests, y déjalos limpios. A partir de aquí, cada ciclo de compilación cuesta entre 5 y 40 minutos y, una vez en revisión, días.

Siguiente: `02-testflight-ios.md` o `03-google-play.md`.
