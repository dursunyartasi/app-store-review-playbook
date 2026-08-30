# Catálogo de rejeições e checklist pré-envio

Isto não é conselho genérico de loja. Cada item aconteceu de verdade ao publicar oito apps, com a causa raiz que acabamos encontrando — que muitas vezes não era o que o aviso de rejeição dizia. Os apps aparecem anonimizados como **App A** (eventos sociais), **App B** (guia de lugares e mapas) e **App C** (ferramenta B2B para agências).

---

## A lição mais cara: o que NÃO escrever nas notas de App Review

O App A, build 49, foi rejeitado pela **diretriz 4.2 (Minimum Functionality)**. O texto da rejeição citou, quase palavra por palavra, as frases que nós mesmos escrevemos nas notas de revisão:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

A Apple leu isso como "isto pertence à distribuição Ad Hoc, não à App Store".

**Regra: nunca enquadre seu app assim** — "small", "niche", "few dozen", "invite-only", "not mass-market", "private group", "closed community", "para uma comunidade específica".

**Enquadramento correto:** "aberto a todos, gratuito, baixável de qualquer cidade ou país; a camada de assinatura curada é *opcional*". Mesmo que o produto seja realmente por convite, ele precisa de uma face que funcione **sem conta**, e as notas devem descrever essa face.

Além disso: **as notas de revisão têm limite de 4000 caracteres.** Ultrapassar faz a API devolver 409.

---

## Checklist pré-envio

Cada caixa vem de uma rejeição. Verifique uma a uma.

### Contas e dados (5.1.1)
- [ ] **Exclusão de conta.** Se dá para criar conta, tem que dar para excluir de dentro do app — 5.1.1(v). *(Causou rejeições em dois dos nossos apps.)* O fluxo que a Apple acabou aceitando cumpria tudo isto, e cada ponto conta:
  - **Imediata e permanente.** Sem desativação, sem período de carência.
  - **Sem passo por suporte, e-mail ou telefone, e sem redirecionamento para a web** — a Apple trata isso como "não excluível de dentro do app".
  - Reautenticação com senha e uma confirmação destrutiva.
  - Uma lista explícita do que será excluído; permissões de terceiros (Instagram/Facebook) revogadas também do lado do provedor.
- [ ] **Strings de propósito das permissões** — 5.1.1(ii). **Armadilha do fluxo bare:** se o diretório `ios/` está no repositório, as strings do `app.json` **não** se propagam para o `Info.plist`. Confira à mão.
- [ ] **Não peça permissões que você não usa.** O App B foi rejeitado por 5.1.1 porque `pickImage` solicitava acesso completo à galeria antes de abrir o seletor. O PHPicker moderno do iOS devolve uma foto **sem permissão nenhuma** → apague a chamada.
- [ ] **Links legais clicáveis** — 2.1(a). O App C foi rejeitado aqui: a frase "ao criar uma conta você aceita os Termos" era **texto simples**, e a tela de login não tinha link nenhum, então quem revisava não conseguiu ver os termos. Abra-os em um **navegador dentro do app** (`expo-web-browser`) em vez de jogar a pessoa no Safari, e coloque também na tela de login.
- [ ] **Tela de contexto prévio de localização** — 5.1.1(iv). Nada de texto enviesado: "Usar minha localização" ❌ → "Continuar" ✅. E não duas opções de "Pular" que confundam.

### Conteúdo gerado por usuários (1.2) — obrigatório se dá para publicar algo
O App A, build 51, foi rejeitado aqui. As quatro são necessárias:
- [ ] Um controle de denúncia **visível**. Um gesto de toque longo **não basta**: quem revisa não vai achar. Coloque um botão "⋯" visível ao lado de cada mensagem, publicação e comentário.
- [ ] Bloqueio (nos dois sentidos: quem foi bloqueado também não pode te escrever).
- [ ] Filtragem de conteúdo em **todos** os endpoints de escrita, não só nos óbvios.
- [ ] Consentimento em mensagens diretas: quem inicia só pode mandar uma mensagem até a outra parte aceitar.

### Sinais de compra (3.1.1) — a armadilha mais traiçoeira em apps gratuitos e B2B
O App C foi rejeitado por esta diretriz **duas vezes**.
- [ ] **Não deixe nenhum preço, nome de plano, contador de créditos, paywall, botão de upgrade ou link externo de compra.** O que derrubou o build 25 foi uma linha dizendo "Intelligence não está no seu plano atual — é preciso Solo ou superior", um contador de créditos e um rótulo de "1 crédito por marca". Só exibir o nome de um plano já basta.
- [ ] **Não se apoie só no argumento 3.1.3(f) "Free Stand-alone Apps": a Apple rejeitou.** Tentamos no build 26.
- [ ] **Uma tela pública de cadastro destrói uma defesa B2B.** Esse era o elo fraco: uma tela de "participe" se lê como venda direta ao consumidor e contradiz o "only sold directly by you to organizations" da 3.1.3(c). A solução no build 27 foi **apagar a tela de cadastro inteira**: o app oferece só login.

### Metadados
- [ ] **Classificação etária** — 2.3.6. Qualquer app com viés de "conhecer pessoas" ou networking precisa de `matureOrSuggestiveThemes` pelo menos em `INFREQUENT_OR_MILD`. Dá para corrigir por API: `PATCH /v1/ageRatingDeclarations/{id}`.
- [ ] **A URL da política de privacidade retorna 200?** Nosso primeiro envio de produção no Play foi rejeitado apenas porque a URL declarada dava 404.

### O que quem revisa vai realmente ver
- [ ] **A conta de demonstração funciona mesmo?** Teste em um aparelho. O App A foi rejeitado por 2.1 porque o login pelo target do Apple Watch nunca havia funcionado: mandava `email` enquanto o backend lia `identifier`, devolvendo 422. Ninguém notou porque o target de celular mandava o campo certo.
- [ ] **Os dados de demo estão frescos?** No App A, 16 dos eventos semeados estavam no passado: quem revisasse abriria um app vazio. Tenha um script idempotente que empurra as datas para a frente.
- [ ] **Um muro de verificação prende quem revisa?** Se alguém cadastrado mas não verificado não vê nada, o app parece fechado. Deixe convidados navegarem; exija verificação só na escrita.
- [ ] **Não publique módulos vazios ou desligados.** Uma feature flag desligada no App A deixava uma seção "Cursos" vazia que causou duas rejeições 2.1 seguidas (App Completeness e depois Information Needed). Acabou removida. **Não envie uma funcionalidade pela metade: tire.**
- [ ] **Você não escolhe o aparelho de revisão.** Para nós vieram respostas de um iPad Air e de um Apple Watch. Teste também os targets que não são o principal.

### Integração com a plataforma (4.0 Design)
- [ ] **Sua função de mapas ou localização está integrada ao app nativo?** O App B foi rejeitado por 4.0.0 por só encaminhar a pessoa ao Google Maps. Ofereça Apple Maps (`maps.apple.com`) como opção.

### Android
- [ ] **A declaração de ID de publicidade bate com o manifesto?** Descompacte o `.aab` e procure `com.google.android.gms.permission.AD_ID`. Se não estiver lá, a declaração deve dizer "não é usado": declaração errada trava o lançamento.
- [ ] **Países de distribuição.** Um dos nossos apps ficou acidentalmente limitado a **um único país** em produção enquanto o iOS distribuía para 175.
- [ ] **Localização em segundo plano** pode acionar a Declaração de Permissão de Localização do Play.
- [ ] **Chave da API de Maps** em `app.json > android.config.googleMaps.apiKey`: sem ela, o `react-native-maps` **quebra na inicialização nativa no Android**. No iOS não acontece (lá o padrão é Apple Maps), e é por isso que passa despercebido.
- [ ] **Google Sign-In precisa de DOIS SHA-1:** sua chave de upload **e** a chave de assinatura do app do Play. O Play gera a dele só depois do primeiro upload do AAB; se esse SHA-1 não for adicionado ao cliente OAuth Android, o Google Sign-In fica quebrado no build do Play. Também não dá para testar em emulador (o SHA-1 de debug não está registrado): é preciso um build assinado pelo Play em um aparelho.

---

## Catálogo de rejeições

| Diretriz | O que a loja disse | Causa raiz real | Solução |
|---|---|---|---|
| **4.2** Minimum Functionality | "small, or niche, set of users" | **Nossa própria frase nas notas de revisão** | Fluxo de descoberta sem conta + endpoints públicos + rebalanceamento de conteúdo |
| **1.2** UGC | sem filtragem / denúncia / bloqueio | A denúncia existia só atrás de um toque longo invisível; ausente em mensagens diretas e salas | Menu "⋯" visível em 8 superfícies, filtro em 9 endpoints de escrita, consentimento em DM |
| **2.1** Conta de demo | não foi possível entrar | O target de Watch mandava `email`, o backend lia `identifier` → 422 | Campo corrigido; seção "WATCH — PLEASE READ FIRST" nas notas |
| **2.1** App Completeness | "could not access the courses" | Feature flag desligada; a seção aparecia vazia | Funcionalidade removida por completo |
| **2.1** Information Needed | "quantos usuários vocês querem atingir?" | O mesmo módulo vazio somado ao enquadramento da 4.2 | Notas reescritas |
| **2.3.6** Classificação etária | "Mature or Suggestive Themes" | Tema de conhecer pessoas não declarado | `PATCH ageRatingDeclarations` pela API do ASC |
| **3.1.1** Compra no app | sinais de compra em um app gratuito | Nome de plano, contador de créditos, texto "é preciso plano Solo" | Removido todo rastro de preço e plano |
| **3.1.1** (segunda vez) | a mesma diretriz de novo | O argumento 3.1.3(f) não bastou; uma **tela pública de cadastro** contradizia a defesa B2B | Tela de cadastro apagada, só login |
| **2.1(a)** App Completeness | não deu para ver os termos | O texto legal era simples, não clicável, e faltava na tela de login | Links clicáveis abrindo em navegador dentro do app |
| **5.1.1(v)** Data Collection | sem exclusão de conta | — | Exclusão de conta dentro do app |
| **5.1.1(ii)** | string de propósito faltando | O fluxo bare não sincroniza `app.json` → `Info.plist` | Editar o `Info.plist` diretamente |
| **5.1.1(iv)** Fluxo de localização | tela prévia enviesada | Texto dos botões e opção dupla de pular | "Continuar", uma única saída |
| **5.1.1** Acesso a fotos | pedia permissão de galeria | O PHPicker não precisava daquela permissão | Chamada de permissão removida |
| **4.0.0** Design | "not integrated with built-in mapping" | Encaminhamento exclusivo ao Google Maps | Opção de Apple Maps adicionada |
| **Play** (produção) | envio rejeitado | A URL declarada de política de privacidade dava 404 | Alias permanente e entrada corrigida no console |

**Nota:** corrigir uma rejeição pode convidar a próxima. Um app passou por quatro rejeições consecutivas e outro por três. Depois de cada correção, revise a lista **inteira**.

---

## Armadilhas da API do App Store Connect e das declarações

Encontradas preenchendo os campos de envio por API:

- **Não existe o tipo de captura `APP_IPHONE_69`.** O maior tipo de iPhone que a API aceita é `APP_IPHONE_67` (1290×2796). Imagens geradas em 1320×2868 para o aparelho de 6,9" são **rejeitadas**: suba as de 6,7" e deixe a Apple escalar.
- **`whatsNew` não pode ser editado em uma primeira versão** — 409, "cannot be edited at this time". Só existe em atualizações.
- **Os tipos de campo da classificação etária são misturados:** alguns BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`) e outros enums STRING (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). O tipo errado devolve 409, e a mensagem de erro informa o conjunto certo.
- **A Apple mudou as faixas etárias em 2025: 12+ não existe mais.** São 4+, 9+, 13+, 16+ e 18+. Respostas honestas podem gerar 4+; suba com `ageRatingOverrideV2` (por exemplo `THIRTEEN_PLUS`).

**Declaração de App Privacy:**
- **Um documento nacional de identidade não é "Sensitive Info".** A categoria sensível da Apple cobre raça, religião, orientação sexual, biometria e afins; documento de identidade não está lá → a caixa certa é **"Other Data Types"**.
- **Dados bancários guardados no seu próprio banco de dados são "Collected".** A Apple só isenta quando o provedor de pagamento os detém e você não consegue acessar.
- ⚠️ **Armadilha do clique às cegas:** o assistente renderiza com alturas diferentes por tipo de dado. Repetir a mesma posição de clique produziu respostas como "O identificador de usuário é usado para rastreamento: SIM" que eram falsas. Verifique com captura o estado final de cada item.

---

## Armadilhas de build e upload

### O trem da versão
**Você não pode subir um build novo contra uma string de versão já aprovada** — erros do altool 90062 / 90186 ("Invalid Pre-Release Train ... closed"). Suba `version` no `app.json` e **recompile**: a string de versão vai dentro do IPA. Queimamos um build inteiro aprendendo isso.

### Upload
- O `eas submit` pode travar (mais de 23 minutos, sem saída) ou falhar com "Failed to authenticate for session". **O caminho confiável é o altool direto:**
  ```bash
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
  ```
  Deixe o `.p8` em `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`. Leva cerca de 15 segundos.
- **"Upload succeeded" não é "aceito".** A Apple ainda pode rejeitar durante o processamento. Consulte até `VALID` e depois expire o build anterior (`PATCH /v1/builds/{id}` com `{"expired": true}`).
- **Targets de Watch e widget precisam de ícone** (`CFBundleIconName`) ou a Apple recusa o upload com o erro **90713**.
- **ITMS-90062** significa "esta versão já está publicada": suba a string de versão.
- **ITMS-90863** (aviso de símbolos do Apple Silicon) é **normal em apps Expo e não causa rejeição.** Não corra atrás disso.

### Ordem do reenvio
1. **Duas versões não podem estar em revisão ao mesmo tempo.** Cancele o `reviewSubmission` existente (`canceled=true`) e espere CANCELING → COMPLETE.
2. A versão vira `DEVELOPER_REJECTED` (editável) → PATCH da string de versão → PATCH da relação com o build.
3. ⚠️ **Armadilha da troca:** logo após o cancelamento, anexar o build devolve 409, e se o seu script seguir mesmo assim, ele envia o **antigo**. Repita o anexo e **verifique o build anexado antes de enviar** (`GET /appStoreVersions/{id}/build`).
4. ⚠️ `POST reviewSubmissionItems` pode devolver 409 `ENTITY_STATE_INVALID` enquanto a transição termina. Funciona segundos depois: torne repetível.

### Ambiente de build local
- **Se o Xcode atualizar no meio da sessão**, os builds falham com "iOS X Platform Not Installed". Solução: `xcodebuild -downloadPlatform iOS` (~8,5 GB, sem sudo) e `xcodebuild -runFirstLaunch`. Ter compilado na mesma manhã não prova que o ambiente continua bom.
- **CocoaPods no Ruby 4.0:** `pod install` morre com `Unicode Normalization not appropriate for ASCII-8BIT`. Rode com `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8`.
- **Modular headers no Podfile:** GoogleSignIn 9.2.0 precisa de `:modular_headers => true` para `AppCheckCore`, `GoogleUtilities` e `RecaptchaInterop`.
- **Um perfil de provisionamento anterior a uma capability nova** faz o build local falhar. O EAS em modo não interativo não atualiza credenciais: ou você renova de forma interativa, ou passa pela API do ASC.
- **As capabilities da Apple Developer podem ser ativadas por API** (sem entrar no portal): `POST /v1/bundleIdCapabilities`. Sem o objeto `settings`, devolve 409.
- **`ANDROID_HOME` é obrigatório** em builds locais de Android; sem ele o Gradle informa "SDK location not found".
- **Nunca edite arquivos-fonte enquanto um archive compila** — o Metro embute um bundle escrito pela metade e o app quebra ao abrir.
- **Os temporários do EAS crescem sem limite** (os nossos chegaram a 35 GB). Limpe; disco cheio faz o build falhar com "No space left".
- Números de build pularem depois de tentativas falhas é normal.

### Erros de publicação no Play
- **"Esta versão não ficará disponível para os usuários atuais…"** → suba o version code, ou publique primeiro por Teste interno ou fechado.
- **"Esta versão não adiciona nem remove nenhum pacote de app."** → o AAB não subiu direito; confira o version code e envie de novo.
- **Os símbolos de debug nativos** devem ir em `native-debug-symbols.zip` com diretórios por ABI (`armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, cada um com `libapp.so`) e sem entradas `__MACOSX` ou `.DS_Store`.
- ⚠️ **Prazos de target API level.** O Play bloqueia a publicação de atualizações para apps que perdem o prazo. Acompanhe.
- **A nuance do AD_ID:** o Firebase Analytics exige a permissão no manifesto e uma declaração de "é usado"; um app sem anúncios não deve ter nenhum dos dois. **A regra é que a declaração bata exatamente com o manifesto**: divergência em qualquer direção trava o lançamento.

### Quebras que só existem em builds standalone
- **Simuladores e dev clients não detectam.** Teste em aparelho real por cabo com `devicectl --console`.
- Se `.env` está no `.gitignore`, ele nunca chega ao archive do EAS: variáveis vazias no bundle e quebra ao abrir. Em um app, *todos* os builds quebravam por isso.
- Um módulo nativo importado dinamicamente que não está instalado é invisível em desenvolvimento (o Metro serve ele) e quebra em standalone com `RCTFatalException: Cannot find module`.
- **O Hermes guarda strings em UTF-16.** Procurar strings não ASCII no bundle como UTF-8 não devolve nada: verifique em UTF-16.

---

## Registro na loja: uma única vez, e manual

- **O registro do app no App Store Connect não pode ser criado por API.** Tentamos e confirmamos. Faça pelo navegador.
- **Criar o app no Play Console também é manual** na primeira vez.
- **O bundle ID fica ligado ao registro permanentemente e não pode ser alterado.**
- **Escolher "Gratuito" no Play é irreversível**: não dá para virar pago depois de publicar.
- ⚠️ **Caracteres não ASCII podem ser descartados no cadastro.** Em uma conta individual da Apple, o nome de desenvolvedor exibido na App Store é o seu nome legal; o nosso perdeu os diacríticos no cadastro. Corrigir por App Store Connect → Business → Legal Entity **não funciona**: esse fluxo te empurra para verificação de endereço e para a cadeia do Paid Apps Agreement, e o nome sozinho não é salvo. O caminho que funciona é Suporte da Apple → "Membership & Account" → correção de nome legal, com verificação de identidade. **Confira a grafia caractere por caractere no cadastro.**

## Limites para um assistente de IA executando este guia

- **Nunca digite a senha nem o código 2FA da Apple ou do Google.** O App Store Connect exige login próprio (a sessão do portal de desenvolvedor não é transferida). O fluxo é: a pessoa entra, confirma, e o assistente segue dali com os passos de API e console.
- **O upload de arquivos por navegador tem limite de 10 MB;** um `.aab` típico passa de 60 MB. Que a pessoa suba, ou automatize com uma conta de serviço do Play e `eas.json > submit.android`.
- **Nunca marque caixas de declaração ou consentimento** sem a aprovação explícita da pessoa.

---

Quando chegar uma rejeição nova, ache primeiro a causa raiz e depois acrescente uma linha aqui. Um guia vale o que o último incidente ensinou a ele.
