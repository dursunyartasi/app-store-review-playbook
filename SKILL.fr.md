---
name: mobile-app-shipping-fr
description: Guide complet pour créer et publier une application mobile sur l'App Store et Google Play, écrit à partir de refus réels et d'échecs de build réels. À utiliser quand la personne veut démarrer un prototype mobile, configurer Expo/React Native, compiler un IPA ou un AAB, envoyer sur TestFlight, envoyer sur Google Play, soumettre à la revue App Store, rédiger les notes de revue, remplir les déclarations du store ou corriger un refus. Se déclenche aussi sur - "application refusée", "Guideline", "App Review", "TestFlight", "eas build", "altool", "profil de provisionnement", "AAB", "Play Console", "App Store Connect", "classification par âge", "déclaration de confidentialité", "captures d'écran".
metadata:
  version: 2.0.0
  source: 8 applications iOS/Android publiées, 2026
---

# Publier une application mobile

Tout ici vient d'applications réellement sorties en boutique, et des refus et échecs de build rencontrés en chemin. Rien n'est reformulé depuis la documentation officielle.

## Commence par savoir où en est la personne

Ne demande que ce que tu ignores encore. Si le message répond déjà à une question, saute-la. **Ne pose jamais plus de cinq questions.**

1. **Que veux-tu faire maintenant ?**
   `nouveau prototype` · `compiler et l'installer sur mon téléphone` · `TestFlight` · `Google Play` · `soumettre à la revue` · `on m'a refusé`
2. **Quelles plateformes ?** iOS, Android, ou les deux.
3. **Les utilisateurs se connectent-ils ou créent-ils du contenu ?** (comptes, publications, commentaires, messages, photos)
4. **As-tu les comptes développeur ?** Apple Developer coûte 99 $ par an et est obligatoire avant que quoi que ce soit atteigne un vrai appareil. Google Play, c'est 25 $ une fois.
5. **Y a-t-il un backend, ou en faut-il un ?**

Puis va au fichier correspondant. Ne lis que celui-là.

| Réponse | Lis |
|---|---|
| nouveau prototype | `references/fr/01-prototype.md` |
| compiler / TestFlight | `references/fr/02-testflight-ios.md` |
| Google Play | `references/fr/03-google-play.md` |
| soumettre à la revue | `references/fr/04-soumission-app-store.md` |
| on m'a refusé | `references/fr/05-refus.md` |
| backend, base de données, e-mail | `references/fr/06-infrastructure.md` |

## Ce que les réponses changent

**La question 3 pèse le plus.** Si les utilisateurs peuvent se connecter, tu dois à Apple une suppression de compte dans l'application (5.1.1(v)), sinon tu seras refusé. Si les utilisateurs peuvent publier quoi que ce soit de visible par d'autres, tu dois les quatre obligations de la 1.2 : signalement visible, blocage, filtrage de contenu et consentement en messagerie privée. Ce sont quatre choses distinctes, et un appui long ne compte pas comme visible. Intègre-les au prototype. Les rajouter après un refus coûte un cycle de revue complet, donc des jours.

**La question 4 conditionne tout le reste.** Sans compte Apple payant, pas de TestFlight, pas d'installation sur appareil au-delà d'un profil gratuit de 7 jours, pas de soumission. Dis-le avant que la personne y passe une journée.

**La question 5 a une réponse peu coûteuse.** `references/fr/06-infrastructure.md` décrit une installation auto-hébergée qui évite la facturation par service : Coolify sur un VPS ordinaire, PostgreSQL et l'offre gratuite de Brevo pour les e-mails transactionnels.

## Règles valables à toutes les étapes

- **Ne saisis jamais le mot de passe Apple ou Google ni le code 2FA de la personne.** App Store Connect exige sa propre connexion, la session du portail développeur ne s'y reporte pas. Demande-lui de se connecter, attends sa confirmation, puis prends en charge les étapes API et console.
- **Ne coche pas les cases de déclaration ou de consentement à sa place.** Ce sont des déclarations juridiques sur son application.
- **« Upload succeeded » ne veut pas dire « accepté ».** Apple refuse aussi pendant le traitement. Interroge jusqu'à ce que le build soit `VALID`.
- **Teste ce que verra la personne qui relit, pas ce que tu vois.** La plupart des refus de `05-refus.md` fonctionnaient parfaitement sur l'appareil et le compte du développeur.
- **Avant de blâmer la revue, vérifie que ce qu'elle n'a pas pu atteindre est vraiment atteignable depuis l'extérieur de ta machine.** Plusieurs refus estampillés d'un numéro de guideline étaient en fait un 404, un feature flag désactivé ou un enregistrement DNS manquant.

## Ordre qu'on ne peut pas intervertir

Certaines étapes ne se réorganisent pas, et se tromper coûte des builds entiers :

1. Monte la **chaîne de version** avant de compiler si l'actuelle est déjà approuvée ou publiée : son train de publication est fermé et l'envoi sera refusé.
2. Vérifie que le **numéro de build est libre** avant de compiler, pas après.
3. Annule toute **soumission en cours de revue** avant d'attacher un nouveau build. Deux versions ne peuvent pas être en revue en même temps.
4. **Vérifie le build attaché** avant de soumettre. Après une annulation, l'appel d'attachement peut échouer pendant que le reste du flux continue et soumet l'ancien binaire.
