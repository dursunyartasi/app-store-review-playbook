# Compilación Android y Google Play

Play es más indulgente que App Review, pero bloquea lanzamientos por papeleo más que por código, y esos bloqueos son fáciles de encontrar.

## Configuración inicial

**Crear la app en Play Console es manual la primera vez.** No hay ruta por API.

Dos decisiones irreversibles:
- **"Gratis" no puede pasar a de pago después de publicar.**
- El nombre del paquete es permanente, como un bundle ID de iOS.

## Firma, y el SHA-1 que pilla a todo el mundo

Tú firmas con una **clave de subida**; Play vuelve a firmar con su propia **clave de firma de app**, que genera solo después de tu primera subida de AAB.

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Mantén el keystore fuera del repositorio. Después:

**Si usas Google Sign-In necesitas AMBAS huellas SHA-1 en el cliente OAuth de Android**: tu clave de subida *y* la clave de firma de app de Play (Play Console → Firma de apps). Si te saltas la segunda, Google Sign-In se rompe específicamente en la build de Play mientras tu build local funciona. Tampoco se puede probar en un emulador, porque el SHA-1 de depuración tampoco está registrado. La prueba funcional exige una build firmada por Play en un dispositivo.

## Compilar

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # o tu ruta del SDK
eas build --platform android --profile production --local --output ./app.aab
```

Sin `ANDROID_HOME`, Gradle informa "SDK location not found".

**Los mapas petarán sin clave.** `react-native-maps` usa Google Maps en Android y **peta en la inicialización nativa** si falta `app.json > android.config.googleMaps.apiKey`. iOS no se ve afectado porque allí Apple Maps es lo predeterminado, que es exactamente por lo que esto llega a producción sin que nadie lo note. Verifica que la clave entró: descomprime el AAB y busca `com.google.android.geo.API_KEY` en el manifiesto.

## Subir

Arrastrar y soltar funciona pero se queda manual para siempre; un AAB típico pasa de 60 MB, por encima de cualquier límite de automatización de navegador. Automatízalo con una cuenta de servicio de Play y `eas.json > submit.android`.

### Errores de publicación que verás

- **"Esta versión no estará disponible para los usuarios actuales porque no les permite actualizar a los paquetes de aplicación recién añadidos."** → sube el version code, o mejor, publica primero por Prueba interna o cerrada.
- **"Esta versión no añade ni elimina ningún paquete de aplicación."** → el AAB no subió bien. Revisa el version code y vuelve a subirlo.
- **Los símbolos de depuración nativos** deben ser un `native-debug-symbols.zip` con directorios por ABI —`armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, cada uno con `libapp.so`— y **sin entradas `__MACOSX` ni `.DS_Store`**.

## Declaraciones que bloquean el lanzamiento

**ID de publicidad.** Descomprime el AAB y busca `com.google.android.gms.permission.AD_ID`. Firebase Analytics exige el permiso y una declaración de "se usa" que coincida; una app sin anuncios no debería tener ninguno de los dos. **La regla es que la declaración coincida exactamente con el manifiesto**: una discrepancia en cualquier dirección bloquea el lanzamiento, y el propio aviso de Play puede confundir sobre qué lado está mal.

**URL de la política de privacidad.** Debe devolver 200. Nuestro primer envío a producción fue rechazado únicamente porque la URL declarada daba 404; no había nada más mal en la app.

**Formulario de seguridad de los datos y cuestionario de clasificación de contenido.** Ambos son obligatorios antes de producción. Respóndelos según lo que la app hace de verdad; se contrastan con los permisos declarados.

**Países de distribución.** Compruébalos. Una de nuestras apps estuvo en producción limitada a **un solo país** mientras iOS se distribuía en 175, que no es un estado que nadie elija a propósito.

## Permisos sensibles

La ubicación en segundo plano y `FOREGROUND_SERVICE_LOCATION` disparan una declaración de permisos de Play que exige un **vídeo de demostración** y una revisión. Si aún no los necesitas, bloquéalos explícitamente en lugar de publicarlos y quedarte atascado:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Añádelos después a propósito, con la declaración y el vídeo preparados.

## Fechas límite del target API level

Play deja de aceptar actualizaciones de apps que incumplen la fecha límite para subir su target API level. La fecha se mueve cada año. **Llévala controlada**: enterarse el día del lanzamiento es un mal día.

## Una nota sobre la velocidad de Play

Play aprueba rápido, y eso corta por los dos lados: una versión rota puede estar publicada en aproximadamente una hora y **no se puede retirar**. La nuestra salió con un fallo en la pantalla de inicio de sesión; el único remedio fue publicar un version code corregido y esperar. Usa Prueba interna primero. Vigila el número de fallos en Play Vitals tras publicar: así confirmamos que la corrección había entrado (10 fallos → 0).
