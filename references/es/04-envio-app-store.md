# Envío a la App Store

La compilación está subida y en `VALID`. Ahora vienen los metadatos, las declaraciones y las notas de revisión, que es donde en realidad se decidieron la mayoría de nuestros rechazos.

## Crea el registro (una vez)

**El registro de la app en App Store Connect no se puede crear por API.** Lo intentamos y lo confirmamos; hazlo en el navegador. El bundle ID queda ligado a ese registro de forma permanente.

Todo lo demás —adjuntar la compilación, metadatos, clasificación por edad, notas de revisión, envío— sí se puede manejar por la API de ASC.

## Capturas de pantalla

- **No existe el tipo de captura `APP_IPHONE_69`.** El mayor que acepta la API es `APP_IPHONE_67` (1290×2796). Las imágenes generadas a 1320×2868 para el dispositivo de 6,9" se **rechazan**. Sube las de 6,7" y deja que Apple las escale.
- `whatsNew` **no se puede editar en una primera versión**: 409, "cannot be edited at this time". Solo existe para actualizaciones.

## Clasificación por edad

- Los tipos de campo están mezclados: unos son BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`) y otros enums STRING (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). El tipo equivocado devuelve 409, y el error nombra el conjunto correcto.
- **Apple cambió las franjas en 2025: 12+ ya no existe.** Son 4+, 9+, 13+, 16+ y 18+.
- Respuestas honestas pueden dar 4+; súbelo con `ageRatingOverrideV2` (por ejemplo `THIRTEEN_PLUS`).
- **Si la app tiene cualquier componente de "conocer gente" o networking, declara `matureOrSuggestiveThemes` al menos como `INFREQUENT_OR_MILD`.** Dejarlo a ninguno fue un rechazo 2.3.6.

## Declaración de App Privacy

- **Un número de documento nacional de identidad no es "Sensitive Info".** La lista de datos sensibles de Apple cubre raza, religión, orientación sexual, biometría y similares; un documento de identidad no está ahí, así que el cajón correcto es **"Other Data Types"**.
- **Los datos bancarios que guardas tú son "Collected".** Apple solo te exime cuando los tiene el proveedor de pagos y tú no puedes acceder a ellos.
- ⚠️ **No hagas clic a ciegas en el asistente.** Se renderiza con alturas distintas según el tipo de dato, así que repetir la misma posición de clic produjo respuestas como "El identificador de usuario se usa para seguimiento: SÍ" que eran sencillamente falsas. Haz captura y verifica el estado final de cada elemento.

## Notas de App Review: el texto con más peso que vas a escribir

Uno de nuestros rechazos vino enteramente de este campo. El rechazo 4.2 de Apple, "small, or niche, set of users", nos citó nuestra propia frase:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**Nunca describas la app como pequeña, de nicho, solo por invitación, cerrada, privada, para una comunidad concreta o no masiva.** Apple lo lee como distribución Ad Hoc, no como App Store.

Escribe en su lugar: abierta a todo el mundo, gratuita, descargable desde cualquier sitio; cualquier capa de curación o membresía es *opcional*. Después describe, en pasos numerados, el recorrido que quien revisa puede hacer **sin cuenta**. Si la app realmente no tiene ese recorrido, constrúyelo antes de enviar: eso es lo que resolvió nuestro 4.2.

**El campo tiene un límite de 4000 caracteres.** Superarlo devuelve 409.

Si la app tiene un objetivo poco habitual (Watch, widget, un flujo específico de dispositivo), pon arriba del todo una sección "PLEASE READ FIRST" con los pasos explícitos de inicio de sesión.

## Cuenta de demostración

Marca "Sign-In Required" y facilita las credenciales.

- **Pruébalas antes en un dispositivo.** Un rechazo 2.1 vino de un inicio de sesión que nunca había funcionado en el target de Watch.
- **Que la cuenta tenga contenido.** En una app, 16 de los 17 eventos sembrados estaban en el pasado, así que quien revisara habría abierto una app vacía. Ten un script idempotente que desplace las fechas de demo hacia adelante y ejecútalo antes de cada envío.
- **Un muro de verificación atrapará a quien revisa.** Si una persona registrada pero sin verificar no ve nada, la app parece cerrada. Deja navegar a los invitados y exige la verificación solo en acciones de escritura.
- **Cierra la cuenta de demo tras la aprobación.** Su contraseña vive en App Store Connect.

## Enlaces legales

Los enlaces de Términos y Privacidad deben ser **pulsables**, abrirse en un navegador dentro de la app en lugar de expulsar a Safari, y aparecer también en la pantalla de **inicio de sesión**, no solo en la de registro. Texto plano no pulsable fue un rechazo 2.1(a): quien revisaba no pudo leer los términos y rechazó solo por eso.

## Si la app es gratuita pero vende algo en algún sitio

La 3.1.1 es la trampa de las apps B2B y gratuitas. **Elimina todo precio, nombre de plan, contador de créditos, muro de pago, botón de mejora y enlace externo de compra.** Un solo nombre de plan bastó para hundir una compilación.

El argumento 3.1.3(f) "Free Stand-alone Apps" **no nos funcionó por sí solo.** El eslabón débil era una pantalla pública de registro: se lee como venta directa al consumidor y contradice el "only sold directly by you to organizations" de la 3.1.3(c). Borramos la pantalla de registro y publicamos solo con inicio de sesión.

## Enviar, y reenviar tras un rechazo

El orden importa. Equivocarse envía el binario incorrecto en silencio.

1. **No puede haber dos versiones en revisión a la vez.** Cancela el `reviewSubmission` existente (`canceled=true`) y espera a CANCELING → COMPLETE.
2. La versión pasa a `DEVELOPER_REJECTED` y es editable. Haz PATCH de la cadena de versión y después de la relación con la compilación.
3. ⚠️ **La trampa del intercambio.** Justo después de cancelar, la llamada que adjunta la compilación devuelve 409. Si tu script sigue igualmente, envía la compilación **antigua**. Reintenta el adjuntado y después **verifica** con `GET /appStoreVersions/{id}/build` antes de enviar. Una vez publicamos la compilación equivocada así.
4. ⚠️ `POST reviewSubmissionItems` puede devolver 409 `ENTITY_STATE_INVALID` mientras termina la transición. Funciona segundos después: hazlo reintentable.

El tipo de publicación es **manual** por defecto: tras la aprobación, alguien todavía tiene que pulsar publicar.

## Cuenta con más de una ronda

Una app pasó por cuatro rechazos consecutivos y otra por tres. Arreglar uno puede destapar el siguiente, y una corrección en un área puede crear un problema en otra. **Después de cada corrección, vuelve a leer la lista completa de `05-rechazos.md`**, no solo el punto que cambiaste.
