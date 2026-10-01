# WATHI Gala Billetterie

Extension WordPress de présentation et de billetterie pour le Gala annuel de WATHI.

Elle permet de présenter l'événement sur une page dédiée, de vendre les billets via des liens de paiement Wave, d'envoyer par e-mail les billets conçus par l'infographiste et d'exporter les ventes réussies et les billets livrés dans un fichier Excel. Une page sécurisée permet à un membre de l'équipe de valider les paiements et d'envoyer les billets.

## Sommaire

1. Fonctionnalités
2. Parcours d'achat
3. Prérequis
4. Installation
5. Configuration
6. Shortcodes
7. Stock de billets (images de l'infographiste)
8. Page d'envoi des billets (espace équipe)
9. Statuts des ventes et des billets
10. Envoi des billets par e-mail
11. Export Excel
12. Rôles et permissions
13. Sécurité
14. Structure de l'extension
15. Base de données
16. Dépannage
17. Évolutions prévues

## 1. Fonctionnalités

**Présentation du gala**
- Page de présentation : titre, date, lieu, programme, intervenants, partenaires, galerie photos
- Compte à rebours jusqu'à la date du gala
- Contenu modifiable depuis l'administration WordPress, sans toucher au code

**Vente de billets**
- Plusieurs catégories de billets (exemple : Standard, VIP, Table entreprise), chacune avec son prix en FCFA et son lien Wave
- Formulaire d'achat : nom, prénom, e-mail, téléphone, catégorie, quantité
- Redirection vers le lien de paiement Wave correspondant
- Places disponibles calculées à partir du stock réel d'images de billets : une catégorie sans billet disponible n'est plus proposée

**Stock de billets**
- Import des images de billets fournies par l'infographiste, par catégorie
- Chaque billet est unique et ne peut être envoyé qu'une seule fois
- Un billet envoyé est automatiquement marqué **Envoyé** et retiré des billets disponibles

**Livraison des billets**
- Attribution automatique d'un billet disponible à chaque vente validée
- Envoi de l'image du billet en pièce jointe par e-mail
- Renvoi possible du même billet au même acheteur en cas de non-réception

**Suivi et export**
- Historique complet des ventes et des billets
- Export Excel (.xlsx) des ventes réussies, des billets livrés et de l'état du stock
- Tableau de bord : billets vendus, recettes, billets restants par catégorie

**Espace équipe**
- Page protégée où un collègue valide les paiements Wave et envoie les billets en un clic

## 2. Parcours d'achat

1. Le visiteur consulte la page du gala et choisit une catégorie de billet.
2. Il remplit le formulaire d'achat. Une vente est créée avec le statut **En attente de paiement** et une référence unique (exemple : `GALA26-0042`).
3. Il est redirigé vers le lien Wave de la catégorie choisie. La référence de la vente lui est affichée et doit être indiquée dans le motif du paiement.
4. Après le paiement, il revient sur une page de confirmation l'informant que son billet lui sera envoyé par e-mail après vérification.
5. Un membre de l'équipe vérifie la réception du paiement sur Wave Business, saisit l'identifiant de transaction et valide la vente.
6. L'extension attribue un billet disponible de la bonne catégorie, l'envoie par e-mail et le marque **Envoyé**. Il disparaît du stock disponible et la vente passe au statut **Billet livré**.

**Mode automatique (optionnel)** : si WATHI dispose d'un accès à l'API Wave Business (Checkout), l'extension peut créer une session de paiement pour chaque vente et recevoir la confirmation par webhook. Les étapes 5 et 6 se font alors sans intervention humaine. Le mode manuel reste disponible en secours.

## 3. Prérequis

- WordPress 6.0 ou plus récent (testé sur WordPress 6.8.10)
- PHP 7.4 ou plus récent (testé sur PHP 7.4.33), avec les extensions `zip`, `xml`, `fileinfo`, `gd` et `mbstring`
- Site servi exclusivement en HTTPS
- Un compte Wave Business avec des liens de paiement (ou un accès API pour le mode automatique)
- Un service d'envoi d'e-mails fiable configuré via SMTP (WP Mail SMTP, Brevo, Mailgun, etc.). L'envoi par défaut de l'hébergeur est déconseillé, les billets risquant d'arriver en spam.
- Une extension d'authentification à deux facteurs pour les comptes de l'équipe (exemple : Two Factor, Wordfence Login Security)
- Composer, pour installer les bibliothèques PHP

Bibliothèque utilisée :

| Bibliothèque | Rôle |
|---|---|
| `phpoffice/phpspreadsheet` (branche 1.29) | Génération des fichiers Excel |

**Compatibilité PHP 7.4**

Le serveur de WATHI fonctionne sous PHP 7.4.33 et ne peut pas être mis à jour pour l'instant. L'extension est donc écrite pour rester compatible avec PHP 7.4 :

- Aucune syntaxe propre à PHP 8 : pas de `match`, d'arguments nommés, de types union, d'opérateur `?->`, de promotion de propriétés dans le constructeur, ni de fonctions comme `str_contains()` ou `str_starts_with()`
- PhpSpreadsheet est fixé sur la branche 1.29, dernière branche compatible PHP 7.4 et qui reçoit encore les correctifs de sécurité (la version 2 exige PHP 8.1)
- Composer est configuré pour simuler PHP 7.4.33, afin qu'aucune dépendance incompatible ne soit installée, même si Composer tourne sur une machine avec une version plus récente

Extrait du `composer.json` :

```json
{
    "require": {
        "php": ">=7.4",
        "phpoffice/phpspreadsheet": "^1.29"
    },
    "config": {
        "platform": {
            "php": "7.4.33"
        },
        "optimize-autoloader": true
    }
}
```

Le code reste compatible avec PHP 8, pour que la future mise à jour du serveur ne demande aucune modification de l'extension.

**Point de vigilance** : PHP 7.4 ne reçoit plus de correctifs de sécurité depuis novembre 2022. L'extension applique ses propres protections (section 13), mais la mise à jour vers PHP 8.2 ou plus récent reste recommandée dès que possible. Avant cela, vérifier la compatibilité du thème et des autres extensions du site sur une copie de test.

## 4. Installation

```bash
cd wp-content/plugins/
git clone <url-du-depot> wathi-gala-billetterie
cd wathi-gala-billetterie
composer install --no-dev --optimize-autoloader
```

Ensuite :

1. Dans l'administration WordPress, aller dans **Extensions** et activer **WATHI Gala Billetterie**.
2. À l'activation, l'extension crée ses tables, le rôle **Gestionnaire billetterie** et le dossier privé de stockage des billets.
3. Aller dans **Gala WATHI > Réglages** pour la configuration.
4. Importer les billets dans **Gala WATHI > Stock de billets**.

Pour une installation sans Composer sur le serveur, générer une archive ZIP contenant le dossier `vendor/` et l'installer via **Extensions > Ajouter > Téléverser**.

## 5. Configuration

Menu **Gala WATHI > Réglages**.

**Onglet Événement**
- Nom et édition du gala (exemple : Gala WATHI 2026)
- Date, heure et lieu
- Description, programme, intervenants, partenaires
- Image de couverture et galerie

**Onglet Billets**
- Catégories : nom, préfixe de numérotation (exemple : `VIP`), prix (FCFA), description, lien de paiement Wave
- Date d'ouverture et de clôture des ventes
- Nombre maximum de billets par commande
- Seuil d'alerte de stock bas (par défaut : 10 billets restants)

**Onglet Paiement Wave**
- Mode : `Manuel` (liens Wave et validation par l'équipe) ou `Automatique` (API Wave Business)
- En mode automatique : clé API, secret du webhook, URL du webhook à déclarer chez Wave (`https://wathi.org/wp-json/wathi-gala/v1/wave-webhook`)
- Délai d'expiration d'une vente non payée (par défaut : 48 heures)

**Onglet E-mails**
- Nom et adresse de l'expéditeur
- Objet et contenu des e-mails (accusé de commande, envoi du billet)
- Adresse en copie cachée pour l'archivage interne
- Adresse de réception des alertes (stock bas, échec d'envoi)
- Bouton **Envoyer un e-mail de test**

## 6. Shortcodes

| Shortcode | Usage |
|---|---|
| `[wathi_gala_presentation]` | Page de présentation complète du gala |
| `[wathi_gala_billets]` | Liste des catégories et formulaire d'achat |
| `[wathi_gala_confirmation]` | Page de retour après paiement |
| `[wathi_gala_envoi]` | Espace équipe : validation des paiements et envoi des billets |

Pages conseillées :

- `/gala` : `[wathi_gala_presentation]` suivi de `[wathi_gala_billets]`
- `/gala/confirmation` : `[wathi_gala_confirmation]`
- `/gala/envoi-billets` : `[wathi_gala_envoi]` (page non indexée et absente du menu)

## 7. Stock de billets (images de l'infographiste)

Les billets ne sont pas générés par l'extension : ils sont conçus par l'infographiste, qui fournit une image par billet. L'extension gère ce stock et garantit qu'un billet n'est jamais envoyé deux fois.

**Format attendu des fichiers**
- Formats acceptés : PNG, JPG ou PDF
- Taille maximale : 5 Mo par billet
- Un fichier par billet, nommé avec le préfixe de la catégorie et un numéro unique : `VIP-001.png`, `VIP-002.png`, `STANDARD-001.png`, etc.
- Chaque image doit afficher visiblement son numéro de billet (et idéalement un QR code contenant ce numéro), pour permettre le contrôle à l'entrée

**Import**
Menu **Gala WATHI > Stock de billets > Importer** (réservé aux administrateurs) :

1. Choisir la catégorie.
2. Téléverser les images une par une, par sélection multiple, ou en une archive ZIP.
3. L'extension vérifie chaque fichier : type réel, taille, nom conforme, numéro non déjà utilisé, image non déjà importée (empreinte SHA-256).
4. Un rapport d'import liste les billets acceptés et les fichiers refusés avec le motif.

Les billets acceptés passent au statut **Disponible**.

**Règles du stock**
- Un billet **Envoyé** n'apparaît plus jamais dans les billets disponibles et ne peut être attribué à aucune autre vente
- Un billet envoyé ne peut être ni supprimé, ni remplacé, ni remis en stock
- Si une vente est annulée après envoi, son billet passe au statut **Invalidé** : il reste hors stock et doit être refusé à l'entrée
- Seul un billet **Disponible** peut être retiré du stock (fichier erroné), avec motif obligatoire
- L'attribution d'un billet est atomique : deux collègues validant une vente au même moment ne peuvent pas recevoir le même billet

**Vue du stock**
Tableau par catégorie : billets disponibles, réservés, envoyés, invalidés. Une alerte e-mail est envoyée lorsque le stock d'une catégorie passe sous le seuil défini, pour demander de nouveaux billets à l'infographiste.

## 8. Page d'envoi des billets (espace équipe)

Cette page permet à un collègue de traiter les ventes sans accéder à l'administration complète de WordPress.

**Accès** : réservé aux utilisateurs connectés ayant le rôle **Gestionnaire billetterie** ou **Administrateur**, avec authentification à deux facteurs. Les autres visiteurs voient un formulaire de connexion. La session expire après 30 minutes d'inactivité.

**Fonctionnement**
- Compteur des billets disponibles par catégorie en haut de page
- Liste des ventes avec filtres par statut, catégorie et date, et recherche par nom, e-mail, téléphone ou référence
- Pour chaque vente en attente : saisie de l'identifiant de transaction Wave et du montant reçu, puis bouton **Valider et envoyer le billet**
- Contrôle automatique : alerte si le montant saisi ne correspond pas au prix attendu, blocage si l'identifiant Wave a déjà été utilisé, blocage si le stock de la catégorie est vide
- Après l'envoi, la vente affiche le numéro du billet attribué (exemple : `VIP-014`) et la date d'envoi
- Bouton **Renvoyer le billet** : renvoie le même billet à la même adresse, sans en consommer un nouveau
- Correction de l'adresse e-mail avant renvoi, enregistrée dans le journal
- Bouton **Annuler la vente** avec motif obligatoire
- Ajout manuel d'une vente (paiement reçu hors site, invitation offerte)
- Le collègue ne voit jamais la liste des fichiers du stock et ne peut pas télécharger les billets disponibles
- Chaque action est enregistrée dans un journal avec le nom de l'utilisateur, la date et l'adresse IP

## 9. Statuts des ventes et des billets

**Ventes**

| Statut | Signification |
|---|---|
| En attente de paiement | Formulaire rempli, paiement non confirmé |
| Payée | Paiement confirmé, billet pas encore envoyé |
| Billet livré | Billet attribué et envoyé avec succès |
| Échec d'envoi | Billet attribué mais l'e-mail n'a pas pu partir |
| Expirée | Aucun paiement reçu dans le délai fixé |
| Annulée | Vente annulée par l'équipe |

Une vente est considérée comme **réussie** lorsqu'elle est au statut Payée, Billet livré ou Échec d'envoi.

**Billets**

| Statut | Signification |
|---|---|
| Disponible | Dans le stock, peut être attribué |
| Réservé | Attribué à une vente, envoi en cours |
| Envoyé | Envoyé à l'acheteur, définitivement retiré du stock |
| Invalidé | Vente annulée après envoi, à refuser à l'entrée |
| Retiré | Fichier erroné retiré du stock avant tout envoi |

En cas d'échec d'envoi, le billet reste attribué à la vente (statut Réservé) pour être renvoyé : il ne retourne pas dans le stock, car il a pu être partiellement transmis.

## 10. Envoi des billets par e-mail

L'e-mail contient :

- Le nom de l'acheteur, la catégorie et le numéro du billet
- La date, l'heure et le lieu du gala
- La référence de la vente
- L'image du billet en pièce jointe (un fichier par billet pour une commande multiple)

Le billet passe au statut **Envoyé** uniquement lorsque le serveur d'envoi a accepté l'e-mail. Si l'envoi échoue, la vente passe en **Échec d'envoi**, une alerte est envoyée à l'équipe et la vente apparaît en tête de liste dans l'espace équipe.

## 11. Export Excel

Disponible dans **Gala WATHI > Export** et dans l'espace équipe.

Le fichier `.xlsx` contient quatre feuilles :

1. **Ventes réussies** : référence, date, nom, prénom, e-mail, téléphone, catégorie, quantité, montant, identifiant de transaction Wave, validé par
2. **Billets livrés** : numéro du billet, référence de la vente, acheteur, e-mail, catégorie, date d'envoi, nombre d'envois, envoyé par
3. **Stock** : numéro du billet, catégorie, statut, date d'import, date d'envoi
4. **Synthèse** : billets vendus, recettes et billets restants par catégorie, total général

Options :
- Filtrer par période ou par catégorie
- Export automatique quotidien envoyé par e-mail à une adresse définie (optionnel)

Le fichier est généré à la demande, envoyé directement au navigateur et jamais conservé dans un dossier public. Nom du fichier : `gala-wathi-ventes-AAAA-MM-JJ.xlsx`

## 12. Rôles et permissions

| Capacité | Administrateur | Gestionnaire billetterie |
|---|---|---|
| Modifier les réglages | Oui | Non |
| Importer des billets | Oui | Non |
| Retirer un billet disponible | Oui | Non |
| Voir les ventes et les compteurs de stock | Oui | Oui |
| Valider et envoyer les billets | Oui | Oui |
| Renvoyer un billet | Oui | Oui |
| Annuler une vente | Oui | Oui |
| Exporter en Excel | Oui | Oui |
| Consulter le journal | Oui | Non |

Pour donner l'accès à un collègue : **Utilisateurs > Ajouter**, choisir le rôle **Gestionnaire billetterie**, puis lui demander d'activer l'authentification à deux facteurs à sa première connexion.

## 13. Sécurité

**Stockage des billets**
- Les images sont stockées dans un dossier privé hors de la médiathèque WordPress : de préférence hors de la racine web, sinon dans `wp-content/uploads/wathi-gala-prive/` protégé par `.htaccess` (Apache) ou par une règle de configuration (Nginx, voir ci-dessous)
- Les fichiers sont renommés avec un identifiant aléatoire : leur nom ne permet pas de deviner l'URL d'un autre billet
- Aucune URL publique ne pointe vers un billet ; l'aperçu dans l'administration passe par un point d'accès authentifié
- L'empreinte SHA-256 de chaque fichier est enregistrée et vérifiée avant l'envoi, pour détecter toute modification

Règle Nginx à ajouter si le serveur n'utilise pas Apache :

```nginx
location ^~ /wp-content/uploads/wathi-gala-prive/ {
    deny all;
}
```

**Import des fichiers**
- Vérification du type réel du fichier (et non de son extension seule)
- Refus de tout fichier qui n'est pas une image ou un PDF, y compris dans les archives ZIP
- Limite de taille par fichier et par import

**Envoi et attribution**
- Attribution des billets par une requête atomique en base de données : un billet ne peut être attribué qu'à une seule vente
- Un identifiant de transaction Wave ne peut être utilisé qu'une seule fois
- Le webhook Wave vérifie la signature de chaque notification avant de modifier une vente

**Accès et formulaires**
- Toutes les actions sont protégées par des nonces WordPress et des vérifications de capacité
- Les données des formulaires sont nettoyées et validées côté serveur
- Protection anti-spam du formulaire d'achat (champ piège et limitation du nombre de soumissions)
- Authentification à deux facteurs et limitation des tentatives de connexion pour les comptes de l'équipe
- Les pages de l'extension sont exclues du cache (extension de cache et Cloudflare)

**Traçabilité**
- Journal non modifiable de toutes les actions : import, attribution, envoi, renvoi, annulation, retrait, export
- Les billets envoyés et leurs enregistrements ne peuvent pas être supprimés depuis l'interface

**Données personnelles**
- Données conservées uniquement pour la durée nécessaire à l'événement, avec une fonction de purge après le gala, conformément à la loi sénégalaise sur la protection des données personnelles (loi n° 2008-12)
- Sauvegarde régulière de la base de données et du dossier privé recommandée pendant la période de vente

## 14. Structure de l'extension

```
wathi-gala-billetterie/
├── wathi-gala-billetterie.php     Fichier principal
├── composer.json
├── uninstall.php                  Nettoyage à la désinstallation
├── includes/
│   ├── class-activator.php        Tables, rôle, dossier privé
│   ├── class-settings.php         Page de réglages
│   ├── class-sales.php            Création et gestion des ventes
│   ├── class-ticket-stock.php     Import et attribution des billets
│   ├── class-private-storage.php  Stockage sécurisé des images
│   ├── class-wave.php             Liens Wave et API Checkout
│   ├── class-webhook.php          Réception des notifications Wave
│   ├── class-mailer.php           Envoi des e-mails
│   ├── class-export.php           Export Excel
│   ├── class-team-page.php        Espace équipe
│   ├── class-logger.php           Journal des actions
│   └── class-shortcodes.php
├── templates/
│   ├── presentation.php
│   ├── formulaire-achat.php
│   ├── confirmation.php
│   ├── espace-equipe.php
│   └── emails/
│       ├── accuse-commande.php
│       ├── envoi-billet.php
│       └── alerte-equipe.php
├── assets/
│   ├── css/
│   ├── js/
│   └── images/
├── languages/
└── vendor/                        Généré par Composer
```

Les gabarits peuvent être surchargés depuis le thème en les copiant dans `wp-content/themes/<theme>/wathi-gala/`.

## 15. Base de données

**`{prefix}wathi_gala_ventes`**

| Colonne | Type | Description |
|---|---|---|
| id | BIGINT | Identifiant |
| reference | VARCHAR(20) | Référence unique (GALA26-0042) |
| nom, prenom | VARCHAR(100) | Acheteur |
| email | VARCHAR(190) | Adresse de livraison du billet |
| telephone | VARCHAR(30) | Numéro de l'acheteur |
| categorie | VARCHAR(50) | Catégorie de billet |
| quantite | INT | Nombre de billets |
| montant | INT | Montant attendu en FCFA |
| statut | VARCHAR(20) | Voir section 9 |
| wave_transaction_id | VARCHAR(100) | Identifiant Wave, unique |
| valide_par | BIGINT | Utilisateur ayant validé |
| date_creation, date_paiement, date_envoi | DATETIME | Horodatages |

**`{prefix}wathi_gala_billets`**

| Colonne | Type | Description |
|---|---|---|
| id | BIGINT | Identifiant |
| numero | VARCHAR(30) | Numéro du billet (VIP-014), unique |
| categorie | VARCHAR(50) | Catégorie |
| fichier | VARCHAR(255) | Nom aléatoire du fichier dans le dossier privé |
| empreinte | CHAR(64) | Empreinte SHA-256, unique |
| statut | VARCHAR(20) | Disponible, Réservé, Envoyé, Invalidé, Retiré |
| vente_id | BIGINT | Vente associée (vide tant que disponible) |
| nombre_envois | INT | Nombre d'envois (renvois compris) |
| importe_par, envoye_par | BIGINT | Utilisateurs concernés |
| date_import, date_envoi | DATETIME | Horodatages |

**`{prefix}wathi_gala_journal`** : historique des actions (vente ou billet concerné, utilisateur, action, adresse IP, date, détail). Table en ajout seul : aucune ligne n'est modifiée ni supprimée par l'extension.

## 16. Dépannage

**Les e-mails n'arrivent pas**
Vérifier la configuration SMTP, utiliser le bouton d'e-mail de test, puis consulter le dossier spam du destinataire. Vérifier aussi la taille des images : une pièce jointe trop lourde peut être refusée.

**Un fichier est refusé à l'import**
Consulter le rapport d'import : nom non conforme, numéro déjà existant, image déjà importée, type ou taille non autorisés.

**Le bouton d'envoi est bloqué**
Le stock de la catégorie est vide. Demander de nouveaux billets à l'infographiste et les importer.

**Les images de billets sont accessibles par URL**
La protection du dossier privé n'est pas active. Vérifier le fichier `.htaccess` ou ajouter la règle Nginx de la section 13.

**L'export Excel ne se télécharge pas**
Vérifier que les extensions PHP `zip` et `xml` sont actives et que le dossier `vendor/` est présent.

**Le webhook Wave ne met pas à jour les ventes**
Vérifier que l'URL du webhook est bien déclarée dans le compte Wave Business, que le secret est correct et que Cloudflare ou un pare-feu ne bloque pas les requêtes vers `/wp-json/wathi-gala/`.

## 17. Évolutions prévues

- Contrôle à l'entrée par recherche ou scan du numéro de billet, avec refus des billets invalidés ou déjà présentés
- Codes promotionnels et tarifs réduits
- Envoi des billets par WhatsApp
- Version anglaise de la page et des e-mails

## Auteur

Développé pour WATHI, le think tank citoyen de l'Afrique de l'Ouest.
[wathi.org](https://wathi.org)
