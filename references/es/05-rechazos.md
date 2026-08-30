# Catálogo de rechazos y lista de comprobación previa al envío

Esto no son consejos genéricos. Cada punto ocurrió de verdad al publicar ocho apps, con la causa raíz que acabamos encontrando, que a menudo no era lo que decía el aviso de rechazo. Las apps aparecen anonimizadas como **App A** (eventos sociales), **App B** (guía de lugares y mapas) y **App C** (herramienta B2B para agencias).

---

## La lección más cara: qué NO escribir en las notas de App Review

La App A, compilación 49, fue rechazada por la **directriz 4.2 (Minimum Functionality)**. El texto del rechazo citaba, casi palabra por palabra, las frases que habíamos escrito nosotros mismos en las notas de revisión:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

Apple lo leyó como "esto pertenece a distribución Ad Hoc, no a la App Store".

**Regla: nunca enmarques tu app así** — "small", "niche", "few dozen", "invite-only", "not mass-market", "private group", "closed community", "para una comunidad concreta".

**Enfoque correcto:** "abierta a todo el mundo, gratuita, descargable desde cualquier ciudad o país; la capa de membresía curada es *opcional*". Aunque el producto sea realmente por invitación, necesita una cara que funcione **sin cuenta**, y las notas deben describir esa cara.

Además: **las notas de revisión tienen un límite de 4000 caracteres.** Al superarlo, la API devuelve 409.

---

## Lista de comprobación previa al envío

Cada casilla viene de un rechazo. Verifícalas una a una.

### Cuentas y datos (5.1.1)
- [ ] **Borrado de cuenta.** Si se pueden crear cuentas, deben poder borrarse desde dentro de la app — 5.1.1(v). *(Provocó rechazos en dos de nuestras apps.)* El flujo que Apple acabó aceptando cumplía todo esto, y cada punto cuenta:
  - **Inmediato y permanente.** Sin desactivación ni periodo de gracia.
  - **Sin paso por soporte, correo o teléfono, y sin redirección a la web** — Apple considera eso "no borrable desde dentro de la app".
  - Reautenticación con contraseña y una confirmación destructiva.
  - Una lista explícita de qué se borra; los permisos de terceros (Instagram/Facebook) revocados también del lado del proveedor.
- [ ] **Cadenas de propósito de los permisos** — 5.1.1(ii). **Trampa del flujo bare:** si el directorio `ios/` está en el repositorio, las cadenas de `app.json` **no** se propagan a `Info.plist`. Revísalo a mano.
- [ ] **No pidas permisos que no uses.** La App B fue rechazada por 5.1.1 porque `pickImage` solicitaba acceso completo a la fototeca antes de abrir el selector. El PHPicker moderno de iOS devuelve una foto **sin ningún permiso** → elimina la llamada.
- [ ] **Enlaces legales pulsables** — 2.1(a). La App C fue rechazada aquí: la frase "al crear una cuenta aceptas los Términos" era **texto plano**, y la pantalla de inicio de sesión no tenía enlaces, así que quien revisaba no pudo ver los términos. Ábrelos en un **navegador dentro de la app** (`expo-web-browser`) en lugar de expulsar a Safari, y ponlos también en la pantalla de inicio de sesión.
- [ ] **Pantalla de contexto previo de ubicación** — 5.1.1(iv). Nada de textos dirigidos: "Usar mi ubicación" ❌ → "Continuar" ✅. Y no dos opciones de "Omitir" que confundan.

### Contenido generado por usuarios (1.2) — obligatorio si se puede publicar algo
La App A, compilación 51, fue rechazada aquí. Hacen falta las cuatro:
- [ ] Un control de denuncia **visible**. Un gesto de pulsación larga **no basta**: quien revisa no lo encontrará. Pon un botón "⋯" visible junto a cada mensaje, publicación y comentario.
- [ ] Bloqueo (en ambos sentidos: quien está bloqueado tampoco puede escribirte).
- [ ] Filtrado de contenido en **todos** los endpoints de escritura, no solo en los evidentes.
- [ ] Consentimiento en mensajes directos: quien inicia solo puede enviar un mensaje hasta que la otra parte acepte.

### Señales de compra (3.1.1) — la trampa más traicionera en apps gratuitas y B2B
La App C fue rechazada por esta directriz **dos veces**.
- [ ] **Que no quede ningún precio, nombre de plan, contador de créditos, muro de pago, botón de mejora ni enlace externo de compra.** Lo que hundió la compilación 25 fue una línea que decía "Intelligence no está en tu plan actual: se requiere Solo o superior", un contador de créditos y una etiqueta de "1 crédito por marca". Basta con mostrar el nombre de un plan.
- [ ] **No te apoyes solo en el argumento 3.1.3(f) "Free Stand-alone Apps": Apple lo rechazó.** Lo intentamos en la compilación 26.
- [ ] **Una pantalla pública de registro destruye una defensa B2B.** Ese era el eslabón débil: una pantalla de "únete" se lee como venta directa al consumidor y contradice el "only sold directly by you to organizations" de la 3.1.3(c). La solución en la compilación 27 fue **borrar la pantalla de registro entera**: la app solo ofrece inicio de sesión.

### Metadatos
- [ ] **Clasificación por edad** — 2.3.6. Cualquier app con un ángulo de "conocer gente" o networking necesita `matureOrSuggestiveThemes` al menos en `INFREQUENT_OR_MILD`. Se corrige por API: `PATCH /v1/ageRatingDeclarations/{id}`.
- [ ] **¿La URL de la política de privacidad devuelve 200?** Nuestro primer envío a producción en Play fue rechazado únicamente porque la URL declarada daba 404.

### Lo que verá realmente quien revisa
- [ ] **¿Funciona de verdad la cuenta de demostración?** Pruébala en un dispositivo. La App A fue rechazada por 2.1 porque el inicio de sesión desde el target de Apple Watch nunca había funcionado: enviaba `email` mientras el backend leía `identifier`, devolviendo 422. Nadie lo notó porque el target de teléfono enviaba el campo correcto.
- [ ] **¿Los datos de demo están frescos?** En la App A, 16 de los eventos sembrados estaban en el pasado: quien revisara habría abierto una app vacía. Ten un script idempotente que adelante las fechas.
- [ ] **¿Un muro de verificación atrapa a quien revisa?** Si alguien registrado pero sin verificar no ve nada, la app parece cerrada. Deja navegar a los invitados; exige verificación solo al escribir.
- [ ] **No publiques módulos vacíos o desactivados.** Un feature flag apagado en la App A dejaba una sección "Cursos" vacía que provocó dos rechazos 2.1 seguidos (App Completeness y luego Information Needed). Acabó eliminada. **No envíes una funcionalidad a medias: quítala.**
- [ ] **No eliges el dispositivo de revisión.** A nosotros nos llegaron respuestas desde un iPad Air y un Apple Watch. Prueba también los targets que no son el principal.

### Integración con la plataforma (4.0 Design)
- [ ] **¿Tu función de mapas o ubicación está integrada con la app nativa?** La App B fue rechazada por 4.0.0 por limitarse a derivar al usuario a Google Maps. Ofrece Apple Maps (`maps.apple.com`) como opción.

### Android
- [ ] **¿La declaración del ID de publicidad coincide con el manifiesto?** Descomprime el `.aab` y busca `com.google.android.gms.permission.AD_ID`. Si no está, la declaración debe decir "no se usa": una declaración equivocada bloquea el lanzamiento.
- [ ] **Países de distribución.** Una de nuestras apps quedó accidentalmente limitada a **un solo país** en producción mientras iOS se distribuía en 175.
- [ ] **La ubicación en segundo plano** puede activar la Declaración de Permisos de Ubicación de Play.
- [ ] **Clave de la API de Maps** en `app.json > android.config.googleMaps.apiKey`: sin ella, `react-native-maps` **peta en la inicialización nativa en Android**. En iOS no pasa nada (allí Apple Maps es lo predeterminado), y por eso se cuela.
- [ ] **Google Sign-In necesita DOS SHA-1:** tu clave de subida **y** la clave de firma de app de Play. Play genera la suya solo tras la primera subida del AAB; si ese SHA-1 no se añade al cliente OAuth de Android, Google Sign-In queda roto en la build de Play. Tampoco se puede probar en emulador (el SHA-1 de depuración no está registrado): hace falta una build firmada por Play en un dispositivo.

---

## Catálogo de rechazos

| Directriz | Qué dijo la tienda | Causa raíz real | Solución |
|---|---|---|---|
| **4.2** Minimum Functionality | "small, or niche, set of users" | **Nuestra propia frase en las notas de revisión** | Flujo de descubrimiento sin cuenta + endpoints públicos + reequilibrio de contenido |
| **1.2** UGC | sin filtrado / denuncia / bloqueo | La denuncia existía solo tras una pulsación larga invisible; ausente en mensajes directos y salas | Menú "⋯" visible en 8 superficies, filtro en 9 endpoints de escritura, consentimiento en DM |
| **2.1** Cuenta de demo | no se pudo iniciar sesión | El target de Watch enviaba `email`, el backend leía `identifier` → 422 | Campo corregido; sección "WATCH — PLEASE READ FIRST" en las notas |
| **2.1** App Completeness | "could not access the courses" | Feature flag apagado; la sección se veía vacía | Funcionalidad eliminada por completo |
| **2.1** Information Needed | "¿a cuántos usuarios apuntáis?" | El mismo módulo vacío más el enfoque de la 4.2 | Notas reescritas |
| **2.3.6** Clasificación por edad | "Mature or Suggestive Themes" | Temática de conocer gente no declarada | `PATCH ageRatingDeclarations` por la API de ASC |
| **3.1.1** Compras integradas | señales de compra en una app gratuita | Nombre de plan, contador de créditos, texto "se requiere plan Solo" | Eliminado todo rastro de precio y plan |
| **3.1.1** (segunda vez) | la misma directriz otra vez | El argumento 3.1.3(f) no bastó; una **pantalla pública de registro** contradecía la defensa B2B | Pantalla de registro borrada, solo inicio de sesión |
| **2.1(a)** App Completeness | no se pudieron ver los términos | El texto legal era plano, no pulsable, y faltaba en la pantalla de inicio de sesión | Enlaces pulsables en navegador dentro de la app |
| **5.1.1(v)** Data Collection | sin borrado de cuenta | — | Borrado de cuenta dentro de la app |
| **5.1.1(ii)** | falta cadena de propósito | El flujo bare no sincroniza `app.json` → `Info.plist` | Editar `Info.plist` directamente |
| **5.1.1(iv)** Flujo de ubicación | pantalla previa dirigida | Textos de botón y doble opción de omitir | "Continuar", una sola salida |
| **5.1.1** Acceso a fotos | pedía permiso de fototeca | El PHPicker no necesitaba ese permiso | Llamada de permiso eliminada |
| **4.0.0** Design | "not integrated with built-in mapping" | Derivación exclusiva a Google Maps | Añadida la opción de Apple Maps |
| **Play** (producción) | envío rechazado | La URL declarada de política de privacidad daba 404 | Alias permanente y entrada corregida en la consola |

**Nota:** arreglar un rechazo puede invitar al siguiente. Una app pasó por cuatro rechazos consecutivos y otra por tres. Después de cada corrección, repasa la lista **entera**.

---

## Trampas de la API de App Store Connect y de las declaraciones

Encontradas rellenando los campos de envío por API:

- **No existe el tipo de captura `APP_IPHONE_69`.** El mayor tipo de iPhone que acepta la API es `APP_IPHONE_67` (1290×2796). Las imágenes generadas a 1320×2868 para el dispositivo de 6,9" se **rechazan**: sube las de 6,7" y deja que Apple escale.
- **`whatsNew` no se puede editar en una primera versión** — 409, "cannot be edited at this time". Solo existe en actualizaciones.
- **Los tipos de campo de la clasificación por edad están mezclados:** unos son BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`) y otros enums STRING (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). El tipo equivocado devuelve 409, y el mensaje de error nombra el conjunto correcto.
- **Apple cambió las franjas de edad en 2025: 12+ ya no existe.** Son 4+, 9+, 13+, 16+ y 18+. Respuestas honestas pueden dar 4+; súbelo con `ageRatingOverrideV2` (por ejemplo `THIRTEEN_PLUS`).

**Declaración de App Privacy:**
- **Un documento nacional de identidad no es "Sensitive Info".** La categoría sensible de Apple cubre raza, religión, orientación sexual, biometría y similares; un documento de identidad no está ahí → el cajón correcto es **"Other Data Types"**.
- **Los datos bancarios que guardas en tu propia base de datos son "Collected".** Apple solo exime cuando los tiene el proveedor de pagos y tú no puedes acceder.
- ⚠️ **Trampa del clic a ciegas:** el asistente se renderiza con alturas distintas según el tipo de dato. Repetir la misma posición de clic produjo respuestas como "El identificador de usuario se usa para seguimiento: SÍ" que eran falsas. Verifica con captura el estado final de cada elemento.

---

## Trampas de compilación y subida

### El tren de la versión
**No puedes subir una compilación nueva contra una cadena de versión ya aprobada** — errores de altool 90062 / 90186 ("Invalid Pre-Release Train ... closed"). Sube `version` en `app.json` y **recompila**: la cadena de versión va dentro del IPA. Quemamos una compilación entera aprendiéndolo.

### Subida
- `eas submit` puede colgarse (más de 23 minutos, sin salida) o fallar con "Failed to authenticate for session". **La vía fiable es altool directamente:**
  ```bash
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
  ```
  Deja el `.p8` en `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`. Tarda unos 15 segundos.
- **"Upload succeeded" no es "aceptado".** Apple todavía puede rechazar durante el procesado. Consulta hasta `VALID` y después caduca la compilación anterior (`PATCH /v1/builds/{id}` con `{"expired": true}`).
- **Los targets de Watch y widget necesitan icono** (`CFBundleIconName`) o Apple rechaza la subida con el error **90713**.
- **ITMS-90062** significa "esta versión ya está publicada": sube la cadena de versión.
- **ITMS-90863** (aviso de símbolos de Apple Silicon) es **normal en apps Expo y no provoca rechazo.** No lo persigas.

### Orden del reenvío
1. **No puede haber dos versiones en revisión a la vez.** Cancela el `reviewSubmission` existente (`canceled=true`) y espera CANCELING → COMPLETE.
2. La versión pasa a `DEVELOPER_REJECTED` (editable) → PATCH de la cadena de versión → PATCH de la relación con la compilación.
3. ⚠️ **Trampa del intercambio:** justo después de cancelar, adjuntar la compilación devuelve 409, y si tu script sigue igualmente, envía la **antigua**. Reintenta el adjuntado y **verifica la compilación adjunta antes de enviar** (`GET /appStoreVersions/{id}/build`).
4. ⚠️ `POST reviewSubmissionItems` puede devolver 409 `ENTITY_STATE_INVALID` mientras termina la transición. Funciona segundos después: hazlo reintentable.

### Entorno de compilación local
- **Si Xcode se actualiza a mitad de sesión**, las compilaciones fallan con "iOS X Platform Not Installed". Solución: `xcodebuild -downloadPlatform iOS` (~8,5 GB, sin sudo) y `xcodebuild -runFirstLaunch`. Que compilara esa misma mañana no prueba que el entorno siga bien.
- **CocoaPods con Ruby 4.0:** `pod install` muere con `Unicode Normalization not appropriate for ASCII-8BIT`. Ejecútalo con `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8`.
- **Modular headers en el Podfile:** GoogleSignIn 9.2.0 necesita `:modular_headers => true` para `AppCheckCore`, `GoogleUtilities` y `RecaptchaInterop`.
- **Un perfil de aprovisionamiento anterior a una capacidad nueva** hace fallar la compilación local. EAS en modo no interactivo no actualiza credenciales: o las renuevas de forma interactiva o pasas por la API de ASC.
- **Las capacidades de Apple Developer se pueden activar por API** (sin entrar al portal): `POST /v1/bundleIdCapabilities`. Sin el objeto `settings` devuelve 409.
- **`ANDROID_HOME` es obligatorio** en compilaciones locales de Android; si no, Gradle informa "SDK location not found".
- **Nunca edites archivos fuente mientras se compila un archive** — Metro empotra un bundle a medio escribir y la app peta al arrancar.
- **Los temporales de EAS crecen sin límite** (los nuestros llegaron a 35 GB). Límpialos; un disco lleno hace fallar la compilación con "No space left".
- Que los números de build salten tras intentos fallidos es normal.

### Errores de publicación en Play
- **"Esta versión no estará disponible para los usuarios actuales…"** → sube el version code, o publica primero por Prueba interna o cerrada.
- **"Esta versión no añade ni elimina ningún paquete de aplicación."** → el AAB no subió bien; revisa el version code y vuelve a subirlo.
- **Los símbolos de depuración nativos** deben ir en `native-debug-symbols.zip` con directorios por ABI (`armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, cada uno con `libapp.so`) y sin entradas `__MACOSX` ni `.DS_Store`.
- ⚠️ **Fechas límite del target API level.** Play bloquea la publicación de actualizaciones a las apps que incumplen la fecha. Llévala controlada.
- **El matiz del AD_ID:** Firebase Analytics exige el permiso en el manifiesto y una declaración de "se usa"; una app sin anuncios no debe tener ninguno de los dos. **La regla es que la declaración coincida exactamente con el manifiesto**: una discrepancia en cualquier dirección bloquea el lanzamiento.

### Fallos que solo existen en compilaciones standalone
- **Los simuladores y los dev clients no los detectan.** Prueba en un dispositivo real por cable con `devicectl --console`.
- Si `.env` está en `.gitignore` nunca llega al archive de EAS: variables vacías en el bundle y fallo al arrancar. En una app petaban *todas* las compilaciones por esto.
- Un módulo nativo importado dinámicamente que no está instalado es invisible en desarrollo (Metro lo sirve) y peta en standalone con `RCTFatalException: Cannot find module`.
- **Hermes guarda las cadenas en UTF-16.** Buscar cadenas no ASCII en el bundle como UTF-8 no devuelve nada: verifica en UTF-16.

---

## Registro en la tienda: una sola vez, y manual

- **El registro de la app en App Store Connect no se puede crear por API.** Lo intentamos y lo confirmamos. Hazlo en el navegador.
- **Crear la app en Play Console también es manual** la primera vez.
- **El bundle ID queda ligado al registro de forma permanente y no se puede cambiar.**
- **Elegir "Gratis" en Play es irreversible**: no se puede pasar a de pago tras publicar.
- ⚠️ **Los caracteres no ASCII pueden perderse en el registro.** En una cuenta individual de Apple, el nombre de desarrollador que aparece en la App Store es tu nombre legal; el nuestro perdió sus diacríticos al registrarse. Corregirlo por App Store Connect → Business → Legal Entity **no funciona**: ese flujo te arrastra a la verificación de dirección y a la cadena del Paid Apps Agreement, y el nombre por sí solo no se guarda. La vía que funciona es Soporte de Apple → "Membership & Account" → corrección de nombre legal, con verificación de identidad. **Revisa la ortografía carácter a carácter al registrarte.**

## Límites para un asistente de IA que ejecute esta guía

- **Nunca escribas la contraseña ni el código 2FA de Apple o Google.** App Store Connect exige su propio inicio de sesión (la sesión del portal de desarrollador no se traslada). El flujo es: la persona entra, lo confirma, y el asistente sigue desde ahí con los pasos de API y consola.
- **La subida de archivos por navegador está limitada a 10 MB;** un `.aab` típico pasa de 60 MB. Que lo suba la persona, o automatízalo con una cuenta de servicio de Play y `eas.json > submit.android`.
- **Nunca marques casillas de declaración o consentimiento** sin la aprobación explícita de la persona.

---

Cuando llegue un rechazo nuevo, encuentra primero la causa raíz y luego añade una fila aquí. Una guía vale lo que le enseñó el último incidente.
