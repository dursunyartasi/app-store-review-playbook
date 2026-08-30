# Do zero a um protótipo rodando

Objetivo: um app rodando no celular da pessoa, construído de modo que os requisitos de loja de `05-rejeicoes.md` já estejam atendidos em vez de remendados depois.

## Decisões antes do primeiro arquivo

**Fluxo managed ou bare.** O managed mantém `ios/` e `android/` como pastas geradas; o bare as coloca no repositório. O bare dá controle nativo e tira isto: **`app.json` deixa de ser a fonte da verdade.** As strings de propósito das permissões não se propagam mais para o `Info.plist`, e a versão passa a vir de `CFBundleShortVersionString` no `Info.plist` mais `MARKETING_VERSION` no pbxproj. As duas coisas já nos custaram rejeições. Comece no managed, a não ser que um módulo nativo obrigue o contrário.

**Identificador de bundle.** Escolha agora e com certeza: assim que existe um registro no App Store Connect, **o bundle ID é permanente.** Use DNS reverso sobre um domínio que você controla.

**O nome que a Apple vai exibir.** Em uma conta individual da Apple Developer, o nome de desenvolvedor mostrado na App Store é o seu nome legal como foi registrado. Caracteres não ASCII podem ser descartados silenciosamente no cadastro (os nossos perderam os diacríticos turcos), e a correção pelo autoatendimento do App Store Connect **não funciona**: ela te empurra para verificação de endereço e para a cadeia do Paid Apps Agreement, e o nome nunca é salvo. Corrigir exige um chamado de suporte com verificação de identidade. **Confira a grafia caractere por caractere durante o cadastro.**

## Estrutura inicial

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # leia o QR com o Expo Go
```

O Expo Go basta até você adicionar um módulo nativo ou precisar de um build assinado. A partir daí é development build ou archive de verdade.

## Já conecte os requisitos de loja

Eles são baratos no primeiro dia e caros depois de uma rejeição.

**Se os usuários podem fazer login — exclusão de conta (5.1.1(v)).** Precisa ser acessível dentro do app, imediata e permanente. Sem desativação, sem período de espera, sem "mande e-mail para o suporte para excluir", sem redirecionamento para um site. Peça a senha de novo, mostre uma confirmação destrutiva, liste o que será excluído e revogue também as permissões de terceiros do lado do provedor.

**Se os usuários podem publicar algo — os quatro requisitos da 1.2.** Um controle visível em cada mensagem, publicação e comentário (um botão "⋯"; o gesto de toque longo é invisível para quem revisa e foi rejeitado), bloqueio funcionando nos dois sentidos, filtragem de conteúdo em **todos** os endpoints de escrita, e um passo de consentimento antes que um desconhecido possa enviar mais de uma mensagem direta.

**Os links legais precisam ser clicáveis (2.1(a)).** "Ao se cadastrar você aceita os Termos" como texto simples é rejeição. Faça links de verdade, abra-os em um navegador dentro do app em vez de jogar a pessoa no Safari, e coloque-os também na tela de login, não só na de cadastro.

**Permissões.** Não peça nada que você não use. Solicitar acesso completo à galeria antes de abrir o seletor foi uma rejeição: o seletor moderno do iOS devolve uma foto sem permissão nenhuma. Telas de contexto prévio não podem usar textos enviesados: "Continuar", não "Usar minha localização".

**Um endereço de contato que realmente receba e-mail.** Se você publica um na ficha da loja ou nas regras dentro do app, o domínio precisa de um registro MX. Veja a armadilha do MX em `06-infraestrutura.md`: o nosso conseguia enviar mas não receber, então o endereço de moderação das nossas regras publicadas não chegava a ninguém.

## Variáveis de ambiente

```
.env            → no .gitignore, como sempre
.easignore      → é ISTO que o EAS lê, e ele substitui o .gitignore
```

**Um `.env` ignorado nunca chega ao archive do EAS.** O bundle sai com variáveis vazias e o app quebra ao abrir — **só** em builds standalone, então o simulador e o dev client parecem perfeitos. Em um dos nossos apps, absolutamente todos os builds quebravam por isso antes de descobrirmos. Ou você configura variáveis de ambiente no EAS, ou garante que o `.easignore` não exclui o que o build precisa.

## Leve para um aparelho real

Um simulador não prova que o app funciona. As quebras que só acontecem em standalone são justamente as que chegam a quem revisa:

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # pega erros de import cedo
```

Depois compile e instale por cabo, e acompanhe o log com `devicectl --console`. Um módulo nativo importado dinamicamente com `import()` que não está instalado é invisível em desenvolvimento — o Metro serve ele — e em standalone quebra com `RCTFatalException: Cannot find module`.

## Antes de seguir adiante

Rode `npx tsc --noEmit` e seus testes, e deixe tudo limpo. Daqui em diante, cada ciclo de build custa de 5 a 40 minutos e, uma vez em revisão, dias.

Próximo: `02-testflight-ios.md` ou `03-google-play.md`.
