# Compilación iOS y TestFlight

El ciclo que repetimos en cada compilación. Entre 15 y 40 minutos; la mayoría de los fallos de abajo cuestan un ciclo entero cuando se pasan por alto.

## Comprueba estas dos cosas antes de compilar

Ambas son baratas de comprobar y caras de descubrir después.

1. **¿Ese número de build ya está usado?** `GET /v1/builds?filter[app]={id}` — un duplicado se rechaza en la subida.
2. **¿La cadena de versión actual ya está publicada?** Si la versión en la App Store está en `READY_FOR_SALE`, su tren de lanzamiento está cerrado y la subida falla con **90186** / **ITMS-90062**. Hay que subir la **cadena de versión**, no solo el número de build, y volver a compilar: la versión va compilada dentro del IPA.

Perdimos cuatro compilaciones en un lanzamiento aprendiendo esto.

## Dónde vive realmente la versión

- **Flujo managed:** `app.json` → `expo.version` y `expo.ios.buildNumber`.
- **Flujo bare:** `ios/<App>/Info.plist` → `CFBundleShortVersionString` y `CFBundleVersion`, más `MARKETING_VERSION` en el pbxproj. **`app.json` se ignora.**

Si editas `app.json` desde un script, parséalo y vuelve a serializarlo (`json.load` / `json.dump`). Leer y escribir sobre el mismo descriptor trunca el archivo: nos costó una compilación.

## Compilar

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # errores de import antes del archive

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

El prefijo `LANG` no es opcional. Con Ruby 4.0 y CocoaPods 1.16, `pod install` muere con `Unicode Normalization not appropriate for ASCII-8BIT`, sobre todo si hay caracteres no ASCII en el proyecto.

`xcodebuild` directo también funciona y es la salida de emergencia cuando las credenciales de EAS están caducadas:

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

Busca `ARCHIVE SUCCEEDED` y después `-exportArchive` con un `ExportOptions.plist` con `method=app-store` y `signingStyle=automatic`.

### Mientras compila

**No edites archivos fuente durante un archive.** Metro empotra un bundle a medio escribir y la app peta al arrancar. No es teórico: nos costó una compilación y parecía un fallo misterioso.

## Subir

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

Deja el `.p8` en `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`; altool lo encuentra ahí. Tarda unos 15 segundos.

**Prefiere esto a `eas submit`.** Hemos visto a `eas submit` colgarse 23 minutos sin salida y sin subir nada, y fallar con "Unable to upload archive. Failed to authenticate for session". altool sí da el error real.

## Después de subir

**`UPLOAD SUCCEEDED` no significa aceptado.** Consulta la API de ASC hasta que la compilación esté en `VALID`; Apple también rechaza durante el procesado. Cuando sea válida, caduca la anterior para que quien prueba solo vea la nueva:

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Errores de subida que conviene reconocer

| Código | Significado |
|---|---|
| **90062** / ITMS-90062 | Esta versión ya está publicada — sube la cadena de versión |
| **90186** | Tren de prelanzamiento cerrado — misma causa |
| **90713** | A un target le falta `CFBundleIconName` — Watch y widgets necesitan su propio icono |
| **ITMS-90863** | Aviso de símbolos de Apple Silicon. **Normal en apps Expo, no es un rechazo.** Ignóralo. |

## Targets adicionales

Los targets de Watch y de Live Activity necesitan sus propios perfiles de aprovisionamiento en `credentials.json`, todos bajo el mismo certificado de distribución. Archivar un target de Watch exige la plataforma **de dispositivo** watchOS en el Mac; el SDK del simulador no basta:

```bash
xcodebuild -downloadPlatform watchOS    # ~4 GB
```

**Prueba los flujos propios del target adicional.** El inicio de sesión de nuestro target de Watch nunca había funcionado —enviaba `email` donde el backend leía `identifier`— y Apple lo encontró antes que nosotros, en un rechazo 2.1.

## Cuando Xcode se actualiza a mitad de proyecto

Una actualización automática deja las compilaciones fallando con `iOS <versión> Platform Not Installed`, aunque esa misma mañana funcionaran:

```bash
xcodebuild -downloadPlatform iOS   # ~8,5 GB, sin sudo
xcodebuild -runFirstLaunch
```

## Mantenimiento

Los temporales de EAS crecen sin límite: los nuestros llegaron a 35 GB en `/var/folders/.../eas-cli-nodejs`. Un disco lleno hace fallar la compilación con `No space left`. Límpialos entre lanzamientos. Que los números de build salten tras intentos fallidos es normal.

Siguiente: `04-envio-app-store.md`.
