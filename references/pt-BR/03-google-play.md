# Build Android e Google Play

O Play é mais tolerante que o App Review, mas bloqueia lançamentos por papelada muito mais do que por código — e é fácil esbarrar nesses bloqueios.

## Configuração inicial

**Criar o app no Play Console é manual na primeira vez.** Não há caminho por API.

Duas escolhas irreversíveis:
- **"Gratuito" não pode virar pago depois da publicação.**
- O nome do pacote é permanente, como um bundle ID do iOS.

## Assinatura, e o SHA-1 que pega todo mundo

Você assina com uma **chave de upload**; o Play reassina com a **chave de assinatura do app** dele, que só é gerada depois do seu primeiro upload de AAB.

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Mantenha a keystore fora do repositório. Depois:

**Se você usa Google Sign-In, precisa das DUAS impressões SHA-1 no cliente OAuth Android**: a sua chave de upload *e* a chave de assinatura do app do Play (Play Console → Assinatura de apps). Se esquecer a segunda, o Google Sign-In quebra especificamente no build do Play enquanto o seu build local funciona. Também não dá para testar em emulador, porque o SHA-1 de debug também não está registrado. O teste funcional exige um build assinado pelo Play em um aparelho.

## Compilar

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # ou o caminho do seu SDK
eas build --platform android --profile production --local --output ./app.aab
```

Sem `ANDROID_HOME`, o Gradle informa "SDK location not found".

**Mapas quebram sem chave.** O `react-native-maps` usa Google Maps no Android e **quebra na inicialização nativa** se faltar `app.json > android.config.googleMaps.apiKey`. O iOS não é afetado porque lá o padrão é o Apple Maps — que é exatamente por que isso chega à produção sem ninguém notar. Confirme que a chave entrou: descompacte o AAB e procure `com.google.android.geo.API_KEY` no manifesto.

## Upload

Arrastar e soltar funciona, mas continua manual para sempre; um AAB típico passa de 60 MB, acima de qualquer limite de automação de navegador. Automatize com uma conta de serviço do Play e `eas.json > submit.android`.

### Erros de publicação que você vai ver

- **"Esta versão não ficará disponível para os usuários atuais porque não permite que eles façam upgrade para os novos pacotes de app adicionados."** → suba o version code, ou melhor, publique primeiro por Teste interno ou fechado.
- **"Esta versão não adiciona nem remove nenhum pacote de app."** → o AAB não subiu direito. Confira o version code e envie de novo.
- **Os símbolos de debug nativos** precisam ser um `native-debug-symbols.zip` com diretórios por ABI — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, cada um com `libapp.so` — e **sem entradas `__MACOSX` ou `.DS_Store`**.

## Declarações que travam o lançamento

**ID de publicidade.** Descompacte o AAB e procure `com.google.android.gms.permission.AD_ID`. O Firebase Analytics exige a permissão e uma declaração de "é usado" correspondente; um app sem anúncios não deve ter nenhum dos dois. **A regra é que a declaração bata exatamente com o manifesto**: divergência em qualquer direção trava o lançamento, e o próprio aviso do Play pode confundir sobre qual lado está errado.

**URL da política de privacidade.** Precisa retornar 200. Nosso primeiro envio de produção foi rejeitado apenas porque a URL declarada dava 404; não havia mais nada errado no app.

**Formulário de segurança dos dados e questionário de classificação de conteúdo.** Os dois são obrigatórios antes da produção. Responda pelo que o app realmente faz; eles são conferidos contra as permissões declaradas.

**Países de distribuição.** Confira. Um dos nossos apps ficou em produção limitado a **um único país** enquanto o iOS distribuía para 175, o que não é um estado que alguém escolha de propósito.

## Permissões sensíveis

Localização em segundo plano e `FOREGROUND_SERVICE_LOCATION` disparam uma declaração de permissão do Play que exige **vídeo de demonstração** e revisão. Se você ainda não precisa delas, bloqueie explicitamente em vez de publicar e ficar travado:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Adicione depois de propósito, com a declaração e o vídeo prontos.

## Prazos de target API level

O Play para de aceitar atualizações de apps que perdem o prazo de subir o target API level. A data muda todo ano. **Acompanhe** — descobrir no dia do lançamento é um dia ruim.

## Uma nota sobre a velocidade do Play

O Play aprova rápido, e isso corta dos dois lados: uma versão quebrada pode estar no ar em cerca de uma hora e **não pode ser retirada**. A nossa saiu com uma quebra na tela de login; o único remédio foi publicar um version code corrigido e esperar. Use Teste interno primeiro. Acompanhe a contagem de falhas no Play Vitals depois de publicar: foi assim que confirmamos que a correção pegou (10 falhas → 0).
