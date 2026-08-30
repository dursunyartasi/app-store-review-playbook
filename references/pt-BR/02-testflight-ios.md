# Build iOS e TestFlight

O ciclo que repetimos a cada build. De 15 a 40 minutos; a maioria das falhas abaixo custa um ciclo inteiro quando passa despercebida.

## Confira estas duas coisas antes de compilar

As duas são baratas de conferir e caras de descobrir depois.

1. **Esse número de build já foi usado?** `GET /v1/builds?filter[app]={id}` — número duplicado é recusado no upload.
2. **A string de versão atual já está publicada?** Se a versão na App Store está em `READY_FOR_SALE`, o trem daquela versão está fechado e o upload falha com **90186** / **ITMS-90062**. É preciso subir a **string de versão**, não só o número de build, e recompilar: a versão fica compilada dentro do IPA.

Perdemos quatro builds em um lançamento aprendendo isso.

## Onde a versão realmente mora

- **Fluxo managed:** `app.json` → `expo.version` e `expo.ios.buildNumber`.
- **Fluxo bare:** `ios/<App>/Info.plist` → `CFBundleShortVersionString` e `CFBundleVersion`, mais `MARKETING_VERSION` no pbxproj. **`app.json` é ignorado.**

Se você editar `app.json` por script, faça parse e serialize de novo (`json.load` / `json.dump`). Ler e escrever no mesmo handle trunca o arquivo — isso nos custou um build.

## Compilar

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # erros de import antes do archive

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

O prefixo `LANG` não é opcional. No Ruby 4.0 com CocoaPods 1.16, o `pod install` morre com `Unicode Normalization not appropriate for ASCII-8BIT`, principalmente se houver caracteres não ASCII no projeto.

`xcodebuild` direto também funciona e é a saída de emergência quando as credenciais do EAS estão desatualizadas:

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

Procure por `ARCHIVE SUCCEEDED` e depois `-exportArchive` com um `ExportOptions.plist` com `method=app-store` e `signingStyle=automatic`.

### Enquanto compila

**Não edite arquivos-fonte durante um archive.** O Metro embute um bundle escrito pela metade e o app quebra ao abrir. Não é teoria: custou um build e parecia uma quebra misteriosa.

## Upload

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

Deixe o `.p8` em `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`; o altool encontra ali. Leva cerca de 15 segundos.

**Prefira isso ao `eas submit`.** Já vimos o `eas submit` travar por 23 minutos sem saída e sem subir nada, e falhar com "Unable to upload archive. Failed to authenticate for session". O altool mostra o erro real.

## Depois do upload

**`UPLOAD SUCCEEDED` não significa aceito.** Consulte a API do ASC até o build ficar `VALID`; a Apple também rejeita durante o processamento. Quando estiver válido, expire o build anterior para que quem testa veja só o novo:

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Erros de upload que vale reconhecer

| Código | Significado |
|---|---|
| **90062** / ITMS-90062 | Esta versão já está publicada — suba a string de versão |
| **90186** | Trem de pré-lançamento fechado — mesma causa |
| **90713** | Um target está sem `CFBundleIconName` — Watch e widgets precisam do próprio ícone |
| **ITMS-90863** | Aviso de símbolos do Apple Silicon. **Normal em apps Expo, não é rejeição.** Ignore. |

## Targets adicionais

Targets de Watch e de Live Activity precisam dos próprios perfis de provisionamento no `credentials.json`, todos sob o mesmo certificado de distribuição. Arquivar um target de Watch exige a plataforma **de dispositivo** watchOS no Mac; o SDK do simulador não basta:

```bash
xcodebuild -downloadPlatform watchOS    # ~4 GB
```

**Teste os fluxos próprios do target adicional.** O login do nosso target de Watch nunca havia funcionado — mandava `email` onde o backend lia `identifier` — e a Apple encontrou antes de nós, em uma rejeição 2.1.

## Quando o Xcode atualiza no meio do projeto

Uma atualização automática deixa os builds falhando com `iOS <versão> Platform Not Installed`, mesmo que na mesma manhã funcionassem:

```bash
xcodebuild -downloadPlatform iOS   # ~8,5 GB, sem sudo
xcodebuild -runFirstLaunch
```

## Manutenção

Os temporários do EAS crescem sem limite: os nossos chegaram a 35 GB em `/var/folders/.../eas-cli-nodejs`. Disco cheio faz o build falhar com `No space left`. Limpe entre lançamentos. Números de build pularem depois de tentativas falhas é normal.

Próximo: `04-envio-app-store.md`.
