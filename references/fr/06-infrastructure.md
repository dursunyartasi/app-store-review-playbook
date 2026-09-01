# L'infrastructure derrière le guide

Le [catalogue des refus](05-refus.md) parle de passer la revue. Ce fichier parle de ce qui tourne en dessous : l'infrastructure sur laquelle ces applications sont publiées, et les pannes rencontrées en l'exploitant.

Même règle que dans le guide : uniquement ce qui est réellement arrivé. C'est **ce que nous utilisons**, pas une affirmation sur ce que tu devrais utiliser.

---

## Ce que nous faisons tourner

| Couche | Choix | Pourquoi |
|---|---|---|
| Mobile | **Expo / React Native** (SDK 57) | Une seule base de code pour les deux stores ; EAS ou builds entièrement locaux |
| Web / API | **Next.js** | Le même TypeScript des deux côtés |
| Base de données | **PostgreSQL**, auto-hébergé | Coût prévisible, pas de mauvaise surprise de tarification à la ligne |
| ORM | **Prisma** | Des migrations qu'on peut relire avant qu'elles touchent la production |
| Fichiers | **MinIO** (compatible S3) ou Cloudflare R2 | Objets auto-hébergés, pas de facture de trafic sortant |
| Hébergement | **Coolify** sur un VPS ordinaire | PaaS auto-hébergé : déploiements git, TLS et conteneurs sans tarif par service |
| E-mail | **Brevo**, offre gratuite via SMTP | 300 e-mails par jour gratuits, largement suffisant longtemps pour l'OTP et les notifications |
| Paiements (Turquie) | **iyzico** | Cartes locales et paiement en plusieurs fois, que Stripe ne couvre pas là-bas |

Une application tourne sur Supabase plutôt que sur PostgreSQL auto-hébergé. Les deux fonctionnent ; ci-dessous, les leçons propres à Supabase sont signalées.

---

## Coolify et le déploiement

Coolify est un PaaS auto-hébergé sur ton propre VPS. Il supprime la facture d'hébergement par service et te rend les pannes d'exploitation qu'une plateforme managée aurait absorbées.

### La pression disque, voilà la panne que tu rencontreras vraiment

Dès que le disque du serveur dépasse environ **80 %**, les déploiements échouent à l'étape d'export des couches alors même que le build a réussi. Coolify l'affiche comme `exit code 255` ou une `DeploymentException` générique — **la vraie cause est masquée.** L'export a besoin d'environ 20 Go libres.

```bash
docker system df           # regarde d'abord
docker builder prune -af   # le cache de build est ce qu'on peut supprimer sans risque
```

Le nettoyage **n'est pas symétrique**, et l'inverser coûte un déploiement :

- `docker image prune -af` — sûr à tout moment.
- `docker builder prune -af` — **jamais juste avant un déploiement.** Il efface
  le cache de build, la couche `apt-get` se relance et retélécharge les paquets ;
  un seul accroc réseau tue alors le build. Hors du chemin de déploiement, c'est
  l'outil le plus efficace dont tu disposes.

Les images sont pour la plupart référencées, les purger libère peu. D'autres
modes de défaillance de Coolify, dont trois causes distinctes produisant une
erreur identique, dans le [manuel d'exploitation Coolify](https://github.com/dursunyartasi/coolify-operations-playbook). **Ne touche pas aux volumes — ce sont les données de ton application.** Lors d'un incident, cela a fait passer le disque de 92 % à 83 % et libéré 7,6 Go ; le déploiement est passé à la nouvelle tentative.

Cette même pression disque se manifeste aussi par un `No such container: <uuid>` passager quand un conteneur auxiliaire meurt en cours de build. Le manque de mémoire produit le même symptôme, alors vérifie les deux.

### Autres comportements de déploiement à connaître
- **Un déploiement recrée tous les services du compose**, pas seulement celui qui a changé — y compris ton conteneur de base de données, dont le **nom change**. Tout ce qui dépend d'un nom de conteneur casse : résous-le à nouveau après chaque déploiement.
- **Un déploiement prend environ 200 à 300 secondes.** Interroge jusqu'à voir le nouveau conteneur et un HTTP 200 ; ne déduis pas le succès de l'appel déclencheur.
- **La première tentative peut échouer sans raison** à l'étape compose. Réessayer suffit généralement, et la production ne tombe pas.
- **Les déploiements ne sont pas déclenchés par webhook** par défaut : c'est une action manuelle ou d'API.
- Si ton VPS est **derrière Cloudflare**, sache que le user agent par défaut d'`urllib` est bloqué. Utilise curl ou définis un user agent de navigateur quand tu scriptes contre ta propre API.

### Notes sur Postgres
- **Supabase / PostgREST :** une table nouvellement créée renvoie `PGRST205 "Could not find the table in schema cache"` alors qu'elle existe. Le cache REST est périmé. Solution : `NOTIFY pgrst, 'reload schema'`.
- **Realtime a besoin de `wal_level=logical`.** Avec la valeur `replica` par défaut, `postgres_changes` s'abonne tranquillement puis ne livre jamais d'événement — une panne silencieuse qui ressemble à un bug côté client. Le changement exige un redémarrage du conteneur, prévois donc une fenêtre de maintenance.

---

## L'e-mail sur l'offre gratuite — et le piège DNS qui a failli tout casser

L'offre gratuite de Brevo (300 e-mails par jour) couvre longtemps l'OTP, les réinitialisations de mot de passe et les notifications. Pointe ton application vers `smtp-relay.brevo.com:587`.

Pour que les e-mails soient délivrés plutôt que jetés, le domaine doit apparaître comme **Authenticated** dans Brevo, ce qui exige :
- **DKIM** — les deux enregistrements CNAME fournis par Brevo
- **DMARC** — commence à `p=none`
- **SPF** — `include:spf.brevo.com`
- L'enregistrement TXT de vérification de Brevo

### ⚠️ Le piège du SPF
Nous avons activé Cloudflare Email Routing pour *recevoir* du courrier sur le même domaine. Cloudflare a proposé d'« ajouter les enregistrements manquants », a vu qu'un enregistrement SPF existait déjà pour Brevo, et a proposé de résoudre le conflit en **supprimant l'enregistrement Brevo**.

Accepter aurait retiré l'authentification à chaque e-mail envoyé par l'application — OTP, notifications, réinitialisations — et les aurait envoyés en spam. La bonne solution est de fusionner les deux includes en **un seul** enregistrement :

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**Un domaine doit avoir exactement un enregistrement SPF.** Plus d'un viole la RFC et casse tout l'envoi. Vérifie avec `dig`, ne te fie pas au panneau.

### Le piège du MX — et pourquoi c'est un problème de store
Ce même domaine **n'avait aucun enregistrement MX**. Il pouvait envoyer, mais pas recevoir. L'adresse de contact de modération que nous avions publiée n'atteignait personne.

Ce n'est pas qu'un bug d'e-mail. La **Guideline 1.2** de l'App Store attend un moyen fonctionnel de signaler du contenu, et nos propres règles promettaient une réponse sous trois jours ouvrés. Une adresse qui jette le courrier en silence, c'est un engagement non tenu et un risque en revue. **Si tu publies une adresse de contact dans la fiche du store ou les règles intégrées, envoie-lui un message de test.**

Autre point : Brevo peut restreindre l'envoi à une liste d'IP autorisées. Ajoute à la fois ta machine de développement et ton serveur, sinon le courrier de production meurt pendant que les tests locaux passent.

---

## Notes sur le build mobile

Les pièges complets de build et d'envoi sont dans le [catalogue](05-refus.md). Les décisions d'infrastructure derrière :

- **Les builds locaux battent EAS distant tant que tu itères.** Les files distantes se remplissent, et EAS non interactif ne met pas à jour les identifiants — un profil de provisionnement antérieur à une nouvelle capability te bloque sans issue. `xcodebuild` local plus `xcrun altool` est la porte de sortie.
- **Pense `.env` du point de vue d'EAS.** Un `.env` ignoré n'atteint jamais l'archive, ce qui donne des variables vides et un plantage au lancement visible uniquement en build standalone.
- **Les builds Android locaux ont besoin de `ANDROID_HOME`**, sinon Gradle signale « SDK location not found ».
- **Automatise l'envoi vers Play avec un compte de service** (`eas.json > submit.android`). Envoyer le `.aab` à la main est l'étape qui reste manuelle le plus longtemps, et l'automatisation navigateur n'aide pas : les fichiers dépassent largement toute limite d'envoi.

---

## D'où vient le VPS

Coolify a besoin d'un VPS ordinaire avec accès root ; aucune plateforme managée n'est nécessaire. N'importe quel hébergeur avec Docker et une IP publique convient. Dimensionnement d'après ce que nous faisons tourner : une petite instance suffit pour l'application, mais **donne au disque plus de marge qu'il ne semble nécessaire**, car la panne d'export de couches décrite plus haut est un problème de disque, pas de CPU. Prévois 20 Go de marge au-delà de tes images.

Les nôtres tournent chez Hostinger. **Lien de parrainage — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — l'utiliser rapporte une commission à l'auteur et te donne une remise. Ce n'est pas une obligation : Coolify fonctionne chez n'importe quel hébergeur avec Docker et un accès root, et rien dans ce guide ne dépend de l'hébergeur.

---

## Le lien avec la revue

Plusieurs refus du catalogue étaient des problèmes d'infrastructure déguisés en numéro de guideline :

| Ça ressemblait à | C'était en fait |
|---|---|
| Guideline 1.2, aucun moyen de signaler du contenu | Une adresse de contact publiée sans enregistrement MX |
| Soumission Play refusée | L'URL déclarée de politique de confidentialité renvoyait 404 |
| 2.1 App Completeness, « l'application plante au lancement » | Le `.env` n'a jamais atteint le build |
| 2.1, « nous n'avons pas pu accéder à la fonctionnalité » | Un feature flag éteint en production |

Avant de blâmer la revue, vérifie si ce qu'elle n'a pas pu atteindre est réellement atteignable depuis l'extérieur de ta machine.
