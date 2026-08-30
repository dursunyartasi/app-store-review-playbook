# Envio para a App Store

O build está no ar e `VALID`. Agora vêm os metadados, as declarações e as notas de revisão — onde a maioria das nossas rejeições foi de fato decidida.

## Crie o registro (uma vez)

**O registro do app no App Store Connect não pode ser criado por API.** Tentamos e confirmamos; faça pelo navegador. O bundle ID fica ligado a esse registro permanentemente.

Todo o resto — anexar o build, metadados, classificação etária, notas de revisão, envio — pode ser conduzido pela API do ASC.

## Capturas de tela

- **Não existe o tipo de captura `APP_IPHONE_69`.** O maior que a API aceita é `APP_IPHONE_67` (1290×2796). Imagens geradas em 1320×2868 para o aparelho de 6,9" são **rejeitadas**. Suba as de 6,7" e deixe a Apple escalar.
- `whatsNew` **não pode ser editado em uma primeira versão** — 409, "cannot be edited at this time". Só existe em atualizações.

## Classificação etária

- Os tipos de campo são misturados: alguns BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`) e outros enums STRING (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). O tipo errado devolve 409, e o erro informa o conjunto correto.
- **A Apple mudou as faixas em 2025: 12+ não existe mais.** São 4+, 9+, 13+, 16+ e 18+.
- Respostas honestas podem gerar 4+; suba com `ageRatingOverrideV2` (por exemplo `THIRTEEN_PLUS`).
- **Se o app tem qualquer viés de "conhecer pessoas" ou networking, declare `matureOrSuggestiveThemes` pelo menos como `INFREQUENT_OR_MILD`.** Deixar em nenhum foi uma rejeição 2.3.6.

## Declaração de App Privacy

- **Um número de documento nacional de identidade não é "Sensitive Info".** A lista de dados sensíveis da Apple cobre raça, religião, orientação sexual, biometria e afins; documento de identidade não está lá, então a caixa certa é **"Other Data Types"**.
- **Dados bancários que você mesmo guarda são "Collected".** A Apple só isenta quando o provedor de pagamento os detém e você não tem acesso.
- ⚠️ **Não clique no assistente às cegas.** Ele renderiza com alturas diferentes por tipo de dado, então repetir a mesma posição de clique produziu respostas como "O identificador de usuário é usado para rastreamento: SIM" que eram simplesmente falsas. Tire captura e verifique o estado final de cada item.

## Notas de App Review — o texto de maior alavancagem que você vai escrever

Uma das nossas rejeições veio inteiramente deste campo. A rejeição 4.2 da Apple, "small, or niche, set of users", citou de volta a nossa própria frase:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**Nunca enquadre o app como pequeno, de nicho, só por convite, fechado, privado, para uma comunidade específica ou não voltado ao grande público.** A Apple lê isso como distribuição Ad Hoc, não App Store.

Escreva em vez disso: aberto a todos, gratuito, baixável de qualquer lugar; qualquer camada curada ou de assinatura é *opcional*. Depois descreva, em passos numerados, o caminho que quem revisa pode percorrer **sem conta**. Se o app realmente não tem esse caminho, construa antes de enviar — foi isso que resolveu a nossa 4.2.

**O campo tem limite de 4000 caracteres.** Ultrapassar devolve 409.

Se o app tem um alvo incomum (Watch, widget, um fluxo específico de aparelho), coloque bem no topo uma seção "PLEASE READ FIRST" com os passos explícitos de login.

## Conta de demonstração

Marque "Sign-In Required" e informe as credenciais.

- **Teste antes em um aparelho.** Uma rejeição 2.1 veio de um login que nunca havia funcionado no target de Watch.
- **Garanta que a conta tenha conteúdo.** Em um app, 16 dos 17 eventos semeados estavam no passado, então quem revisasse abriria um app vazio. Tenha um script idempotente que empurra as datas de demo para a frente e rode antes de cada envio.
- **Um muro de verificação vai prender quem revisa.** Se alguém cadastrado mas não verificado não vê nada, o app parece fechado. Deixe convidados navegarem; exija verificação só em ações de escrita.
- **Feche a conta de demo depois da aprovação.** A senha dela fica no App Store Connect.

## Links legais

Os links de Termos e Privacidade precisam ser **clicáveis**, abrir em um navegador dentro do app em vez de jogar a pessoa no Safari, e aparecer também na tela de **login**, não só na de cadastro. Texto simples não clicável foi uma rejeição 2.1(a): quem revisava não conseguiu ler os termos e rejeitou só por isso.

## Se o app é gratuito mas vende algo em algum lugar

A 3.1.1 é a armadilha dos apps B2B e gratuitos. **Remova todo preço, nome de plano, contador de créditos, paywall, botão de upgrade e link externo de compra.** Um único nome de plano bastou para derrubar um build.

O argumento 3.1.3(f) "Free Stand-alone Apps" **não funcionou sozinho para nós.** O elo fraco era uma tela pública de cadastro: ela se lê como venda direta ao consumidor e contradiz o "only sold directly by you to organizations" da 3.1.3(c). Apagamos a tela de cadastro e publicamos só com login.

## Enviar, e reenviar depois de uma rejeição

A ordem importa. Errar envia o binário errado em silêncio.

1. **Duas versões não podem estar em revisão ao mesmo tempo.** Cancele o `reviewSubmission` existente (`canceled=true`) e espere CANCELING → COMPLETE.
2. A versão vira `DEVELOPER_REJECTED` e fica editável. Faça PATCH da string de versão e depois da relação com o build.
3. ⚠️ **A armadilha da troca.** Logo após o cancelamento, a chamada que anexa o build devolve 409. Se o seu script seguir mesmo assim, ele envia o build **antigo**. Repita o anexo e depois **verifique** com `GET /appStoreVersions/{id}/build` antes de enviar. Já publicamos o build errado assim uma vez.
4. ⚠️ `POST reviewSubmissionItems` pode devolver 409 `ENTITY_STATE_INVALID` enquanto a transição termina. Funciona segundos depois: torne o passo repetível.

O tipo de publicação é **manual** por padrão: depois da aprovação, alguém ainda precisa apertar publicar.

## Conte com mais de uma rodada

Um app passou por quatro rejeições consecutivas e outro por três. Corrigir uma pode expor a seguinte, e uma correção em uma área pode criar problema em outra. **Depois de cada correção, releia a lista inteira em `05-rejeicoes.md`**, não só o item que você mudou.
