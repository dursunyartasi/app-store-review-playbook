# A infraestrutura por trás do guia

O [catálogo de rejeições](05-rejeicoes.md) trata de passar na revisão. Este arquivo trata do que roda por baixo: a infraestrutura sobre a qual esses apps são publicados e as falhas que encontramos operando ela.

Mesma regra do guia: só coisas que aconteceram. Isto é **o que nós usamos**, não uma afirmação sobre o que você deveria usar.

---

## O que rodamos

| Camada | Escolha | Por quê |
|---|---|---|
| Mobile | **Expo / React Native** (SDK 57) | Um só código para as duas lojas; EAS ou build totalmente local |
| Web / API | **Next.js** | O mesmo TypeScript nas duas pontas |
| Banco de dados | **PostgreSQL**, auto-hospedado | Custo previsível, sem surpresa de preço por linha |
| ORM | **Prisma** | Migrações que podemos revisar antes de tocarem a produção |
| Arquivos | **MinIO** (compatível com S3) ou Cloudflare R2 | Objetos auto-hospedados, sem conta de saída de dados |
| Hospedagem | **Coolify** em um VPS comum | PaaS auto-hospedado: deploy por git, TLS e contêineres sem preço por serviço |
| E-mail | **Brevo**, plano gratuito por SMTP | 300 e-mails por dia grátis, suficiente para OTP e notificações por muito tempo |
| Pagamentos (Turquia) | **iyzico** | Cartões locais e parcelamento, que o Stripe não cobre lá |

Um dos apps roda sobre Supabase em vez de PostgreSQL auto-hospedado. Os dois funcionam; abaixo está marcado quais lições são específicas do Supabase.

---

## Coolify e deploy

O Coolify é um PaaS auto-hospedado no seu próprio VPS. Ele elimina a conta de hospedagem por serviço e entrega para você as falhas operacionais que uma plataforma gerenciada teria absorvido.

### A pressão de disco é a falha que você vai enfrentar de verdade

Assim que o disco do servidor passa de cerca de **80%**, os deploys falham na fase de exportação de camadas mesmo que o build em si tenha terminado bem. O Coolify mostra isso como `exit code 255` ou um `DeploymentException` genérico: **a causa real fica escondida.** A exportação precisa de algo em torno de 20 GB livres.

```bash
docker system df           # olhe primeiro
docker builder prune -af   # o cache de build é o que dá para apagar com segurança
```

O cache de build pode ser limpo sem problema (o próximo build fica um pouco mais lento). As imagens costumam estar referenciadas, então podá-las libera pouco. **Não toque nos volumes: são os dados da sua aplicação.** Em um incidente isso levou o disco de 92% para 83% e liberou 7,6 GB; o deploy funcionou na nova tentativa.

Essa mesma pressão de disco também aparece como um `No such container: <uuid>` transitório quando um contêiner auxiliar morre no meio do build. Pressão de memória produz o mesmo sintoma, então verifique os dois.

### Outros comportamentos de deploy que vale conhecer
- **Um deploy recria todos os serviços do compose**, não só o que mudou, incluindo o seu contêiner de banco de dados, cujo **nome muda**. Tudo que depende de um nome de contêiner quebra: resolva de novo depois de cada deploy.
- **Um deploy leva de 200 a 300 segundos.** Consulte até ver o contêiner novo e um HTTP 200; não considere sucesso pela chamada que dispara.
- **A primeira tentativa pode falhar sem motivo** na fase de compose. Repetir costuma funcionar e a produção não cai.
- **Deploys não são disparados por webhook** por padrão: são ação manual ou de API.
- Se o seu VPS está **atrás do Cloudflare**, saiba que o user agent padrão do `urllib` é bloqueado. Use curl ou defina um user agent de navegador quando programar contra a sua própria API.

### Notas sobre Postgres
- **Supabase / PostgREST:** uma tabela nova devolve `PGRST205 "Could not find the table in schema cache"` mesmo existindo. O cache REST está desatualizado. Solução: `NOTIFY pgrst, 'reload schema'`.
- **Realtime precisa de `wal_level=logical`.** Com o valor padrão `replica`, o `postgres_changes` se inscreve tranquilamente e depois nunca entrega um evento: uma falha silenciosa que parece bug de cliente. Mudar exige reiniciar o contêiner, então reserve uma janela de manutenção.

---

## E-mail no plano gratuito, e a armadilha de DNS que quase quebrou tudo

O plano gratuito do Brevo (300 e-mails por dia) cobre OTP, redefinições de senha e notificações por muito tempo. Aponte seu app para `smtp-relay.brevo.com:587`.

Para os e-mails caírem na caixa de entrada em vez do lixo, o domínio precisa aparecer como **Authenticated** no Brevo, o que exige:
- **DKIM** — os dois registros CNAME que o Brevo fornece
- **DMARC** — comece em `p=none`
- **SPF** — `include:spf.brevo.com`
- O registro TXT de verificação do Brevo

### ⚠️ A armadilha do SPF
Ativamos o Cloudflare Email Routing para *receber* e-mail no mesmo domínio. O Cloudflare se ofereceu para "adicionar os registros faltantes", viu que já existia um registro SPF do Brevo e propôs resolver o conflito **apagando o registro do Brevo**.

Aceitar isso teria tirado a autenticação de todos os e-mails que o app envia — OTP, notificações, redefinições de senha — e os teria mandado para spam. A solução é juntar os dois includes em **um único** registro:

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**Um domínio deve ter exatamente um registro SPF.** Mais de um viola a RFC e quebra todo o envio. Verifique com `dig`, não confie no painel.

### A armadilha do MX, e por que isso é um problema de loja
Esse mesmo domínio **não tinha registro MX nenhum**. Conseguia enviar, mas não receber. O endereço de contato de moderação que havíamos publicado não chegava a ninguém.

Isso não é só um bug de e-mail. A **diretriz 1.2** da App Store espera um caminho funcional para denunciar conteúdo, e as nossas próprias regras prometiam resposta em até três dias úteis. Um endereço que descarta e-mail em silêncio é um compromisso quebrado e um risco na revisão. **Se você publica um endereço de contato na ficha da loja ou nas regras dentro do app, mande uma mensagem de teste para ele.**

Outra coisa: o Brevo pode restringir o envio a uma lista de IPs autorizados. Adicione tanto a sua máquina de desenvolvimento quanto o seu servidor, ou o e-mail de produção morre enquanto os testes locais passam.

---

## Notas de build mobile

As armadilhas completas de build e upload estão no [catálogo](05-rejeicoes.md). As decisões de infraestrutura por trás delas:

- **Builds locais ganham do EAS remoto quando você está iterando.** As filas remotas enchem, e o EAS em modo não interativo não atualiza credenciais, então um perfil de provisionamento anterior a uma capability nova te trava sem saída. `xcodebuild` local mais `xcrun altool` é a rota de fuga.
- **Pense no `.env` do ponto de vista do EAS.** Um `.env` ignorado nunca chega ao archive, o que gera variáveis vazias e uma quebra ao abrir que só aparece em builds standalone.
- **Builds locais de Android precisam de `ANDROID_HOME`** ou o Gradle informa "SDK location not found".
- **Automatize o upload para o Play com uma conta de serviço** (`eas.json > submit.android`). Subir o `.aab` na mão é o passo que fica manual por mais tempo, e a automação de navegador não ajuda: os arquivos passam de qualquer limite de upload.

---

## De onde vem o VPS

O Coolify precisa de um VPS comum com acesso root; não é preciso plataforma gerenciada. Qualquer provedor com Docker e um IP público serve. Dimensionando pelo que rodamos: uma instância pequena basta para o app, mas **dê ao disco mais folga do que parece necessário**, porque a falha de exportação de camadas descrita acima é problema de disco, não de CPU. Reserve 20 GB de folga além das suas imagens.

Os nossos rodam na Hostinger. **Link de indicação — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — usá-lo rende uma comissão ao autor e um desconto a você. Não é obrigatório: o Coolify roda em qualquer provedor com Docker e acesso root, e nada neste guia depende da hospedagem.

---

## Como isso se liga à revisão

Várias das rejeições do catálogo eram problemas de infraestrutura vestindo um número de diretriz:

| Parecia | Na verdade era |
|---|---|
| Diretriz 1.2, sem forma de denunciar conteúdo | Um endereço de contato publicado sem registro MX |
| Envio ao Play rejeitado | A URL declarada de política de privacidade dava 404 |
| 2.1 App Completeness, "o app quebra ao abrir" | O `.env` nunca chegou ao build |
| 2.1, "não conseguimos acessar o recurso" | Uma feature flag desligada em produção |

Antes de culpar quem revisa, verifique se aquilo que a pessoa não conseguiu alcançar é realmente alcançável de fora da sua máquina.
