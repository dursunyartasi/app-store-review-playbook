---
name: mobile-app-shipping-pt-br
description: Guia completo para criar e publicar um app móvel na App Store e no Google Play, escrito a partir de rejeições reais e falhas de build reais. Use quando a pessoa quiser começar um protótipo móvel, configurar Expo/React Native, compilar um IPA ou AAB, subir para o TestFlight, subir para o Google Play, enviar para revisão da App Store, escrever as notas de revisão, preencher declarações da loja ou resolver uma rejeição. Também dispara com - "app rejeitado", "Guideline", "App Review", "TestFlight", "eas build", "altool", "perfil de provisionamento", "AAB", "Play Console", "App Store Connect", "classificação etária", "declaração de privacidade", "capturas de tela".
metadata:
  version: 2.0.0
  fonte: 8 apps iOS/Android publicados, 2026
---

# Publicando um app móvel

Tudo aqui vem de apps que chegaram às lojas de verdade, e das rejeições e falhas de build encontradas no caminho. Nada foi parafraseado da documentação oficial.

## Comece descobrindo onde a pessoa está

Pergunte só o que ainda falta. Se a mensagem já responde a algo, pule. **Nunca faça mais de cinco perguntas.**

1. **O que você quer fazer agora?**
   `protótipo novo` · `compilar e colocar no meu celular` · `TestFlight` · `Google Play` · `enviar para revisão` · `fui rejeitado`
2. **Quais plataformas?** iOS, Android ou as duas.
3. **Os usuários fazem login ou criam conteúdo?** (contas, publicações, comentários, mensagens, fotos)
4. **Você tem as contas de desenvolvedor?** A Apple Developer custa US$ 99 por ano e é obrigatória antes de qualquer coisa chegar a um aparelho real. O Google Play são US$ 25 uma única vez.
5. **Existe um backend, ou vai precisar de um?**

Depois vá ao arquivo correspondente. Leia só ele.

| Resposta | Leia |
|---|---|
| protótipo novo | `references/pt-BR/01-prototipo.md` |
| compilar / TestFlight | `references/pt-BR/02-testflight-ios.md` |
| Google Play | `references/pt-BR/03-google-play.md` |
| enviar para revisão | `references/pt-BR/04-envio-app-store.md` |
| fui rejeitado | `references/pt-BR/05-rejeicoes.md` |
| backend, banco de dados, e-mail | `references/pt-BR/06-infraestrutura.md` |

## O que as respostas mudam

**A pergunta 3 é a que mais pesa.** Se os usuários podem fazer login, você deve à Apple a exclusão de conta dentro do app (5.1.1(v)), ou será rejeitado. Se os usuários podem publicar qualquer coisa visível para outros, você deve as quatro obrigações da 1.2: denúncia visível, bloqueio, filtragem de conteúdo e consentimento em mensagens diretas. São quatro coisas distintas, e um gesto de toque longo não conta como visível. Construa isso no protótipo. Adicionar depois de uma rejeição custa um ciclo inteiro de revisão, ou seja, dias.

**A pergunta 4 é o portão de tudo.** Sem uma conta paga da Apple não existe TestFlight, nem instalação em aparelho além de um perfil gratuito de 7 dias, nem envio. Diga isso antes que a pessoa gaste um dia compilando.

**A pergunta 5 tem uma resposta barata.** `references/pt-BR/06-infraestrutura.md` descreve uma configuração auto-hospedada que evita cobrança por serviço: Coolify em um VPS comum, PostgreSQL e o plano gratuito do Brevo para e-mail transacional.

## Regras que valem em todas as etapas

- **Nunca digite a senha nem o código 2FA da Apple ou do Google da outra pessoa.** O App Store Connect exige login próprio e a sessão do portal de desenvolvedor não é transferida. Peça que ela entre, espere a confirmação e então conduza os passos de API e console.
- **Não marque caixas de declaração ou consentimento por ela.** São declarações legais sobre o app dela.
- **"Upload succeeded" não significa "aceito".** A Apple também rejeita durante o processamento. Consulte até o build ficar `VALID`.
- **Teste o que quem revisa vai ver, não o que você vê.** A maioria das rejeições em `05-rejeicoes.md` eram coisas que funcionavam no aparelho e na conta de quem desenvolvia.
- **Antes de culpar quem revisa, verifique se aquilo que a pessoa não conseguiu alcançar é alcançável de fora da sua máquina.** Várias rejeições por diretriz eram, na verdade, um 404, uma feature flag desligada ou um registro DNS faltando.

## Ordem que não pode ser alterada

Alguns passos não admitem reordenação, e errar custa builds inteiros:

1. Suba a **string de versão** antes de compilar se a atual já estiver aprovada ou publicada: o trem daquela versão está fechado e o upload será recusado.
2. Confira que o **número de build está livre** antes de compilar, não depois.
3. Cancele qualquer **envio em revisão** antes de anexar um build novo. Duas versões não podem estar em revisão ao mesmo tempo.
4. **Verifique o build anexado** antes de enviar. Depois de um cancelamento, a chamada de anexar pode falhar enquanto o resto do fluxo continua e envia o binário antigo.
