# WhatsApp IA — Module Agentia (agent WhatsApp Business pour TPE/artisans FR)

> Spécifications d'intégration à l'usage de l'équipe Agentia.
> Livrable commercial associé : `index.html` (landing) · `script-pilotage.txt` (vente téléphonique).
> Ne pas publier ce dépôt sur GitHub (règle interne Agentia).

---

## 1. L'offre (rappel)

| Offre | Prix | Contenu |
|---|---|---|
| Essai gratuit | 0 € — 14 jours | Réponses IA aux questions fréquentes + horaires. Installation offerte le temps de l'essai. |
| Standard | 49 €/mois | Réponses IA aux questions fréquentes + horaires + prise de rendez-vous. |
| Pro (le plus choisi) | 99 €/mois | Tout Standard + rappels de RDV automatiques + prise de commande simple + transfert humain + alertes urgentes. |
| Installation | 199 € (unique) | Setup API Meta (numéro dédié, validation Meta Business, templates), configuration des règles IA, formation 1 h. Offerte pendant l'essai ; facturée au passage en offre payante. |

**À savoir** : les frais par conversation facturés par Meta (section 5) ne sont **jamais inclus** dans ces prix — ils sont réglés directement par le client à Meta. C'est une promesse commerciale de la landing, à tenir en interne.

---

## 2. Architecture cible

```
Client WhatsApp ──> Numéro dédié (WABA) ──> Meta Cloud API (webhook) ──> Service Agentia
                                                                          │
                                            ┌─────────────────────────────┤
                                            ▼                             ▼
                                   Moteur de règles IA            Tableau de bord
                                   (FAQ, horaires, tarifs,   (conversations, transferts,
                                   RDV, commandes)             statistiques, réglages)
                                            │
                                            ▼
                                   Transfert humain ──> Application WhatsApp
                                   (reprise en main          Business du client
                                    par l'artisan)            (son téléphone)
```

- **API retenue** : **Meta Cloud API** (hébergée par Meta, gratuite d'hébergement). Pas d'On-Premises (serveur + certificats + maintenance : non pertinent pour une TPE).
- **Fournisseur (BSP)** : aucun — branchement direct sur la Cloud API (pas de surcoût type Twilio/360dialog). Le client ne paie que les frais de conversation Meta.
- **Numéro** : dédié (recommandé) ou migration du numéro existant (voir § 3).
- **Moteur IA** : LLM avec *prompt système* généré à partir des règles du client (horaires, tarifs, FAQ, conditions) + garde-fous (mots-clés de transfert, compteur de tentatives, demandes explicites d'humain). Aucun appel LLM sans contexte : l'historique de la conversation est passé au modèle.
- **Fenêtre de 24 h** (règle Meta) : les réponses libres (non modèles) ne sont possibles que **dans les 24 h suivant le dernier message du client**. Au-delà, seuls les **templates approuvés** peuvent être envoyés (payants). C'est ce qui justifie les rappels de RDV « utilitaires » du Pro.

---

## 3. Étapes de branchement (checklist technique)

### Phase A — Préparation (agent Agentia, ~1 h)
1. [ ] Récupérer les justificatifs du client : **SIRET / extrait K-bis / RCS**, site web ou page professionnelle, pièce d'identité du gérant.
2. [ ] Choisir le **numéro dédié** (recommandé) : sim ou VoIP française, **jamais utilisé** par l'application WhatsApp classique ou WhatsApp Business. Coût indicatif : 5–15 €/mois selon opérateur (à la charge du client, hors offre).
   - Si le client tient à son numéro actuel : prévoir la **migration** (le numéro doit être supprimé de l'app WhatsApp existante avant l'enregistrement WABA). Attention au temps mort pendant la bascule.
3. [ ] Créer/prendre la main sur le **compte Meta Business** du client (`business.facebook.com`) — ou créer le compte à partir de zéro avec les documents du client.
4. [ ] Vérifier que le **nom d'affichage** (ex. « Plomberie Martin ») correspond à l'activité déclarée (règle Meta : le nom doit être identifiable par le client).

### Phase B — Validation Meta Business (client + Agentia)
5. [ ] Déclencher la **vérification de l'entreprise** dans l'Espace Business : documents d'identité + justificatif d'activité. Délai typique : **2 à 10 jours ouvrés** (souvent ~48 h pour les micro-entreprises avec SIRET propre).
6. [ ] (Optionnel mais recommandé pour la crédibilité) Vérification « entreprise officielle » (pastille verte) — non requise pour démarrer.

### Phase C — Création de l'app et du WABA (agent Agentia, ~2 h)
7. [ ] Créer l'**application Meta** (type *Business*) sur `developers.facebook.com`, ajouter le produit **WhatsApp**.
8. [ ] Créer le **WABA** (WhatsApp Business Account) depuis l'Espace Business, associé au numéro dédié.
9. [ ] **Enregistrer le numéro** : vérification par SMS ou appel (capacité de réception requise).
10. [ ] Configurer le **webhook** : notre endpoint reçoit les événements `messages` et `message_status` (HTTPS, token de vérification).
11. [ ] Générer un **token d'accès permanent** (System User + permissions `whatsapp_business_messaging`, `whatsapp_business_management`) — stocké dans le coffre de secrets Agentia, jamais dans le code ou le repo.

### Phase D — Templates de messages (agent Agentia)
12. [ ] Créer les **templates** (voir § 4) dans la console WABA, langue `fr`, catégories adaptées (UTILITY / MARKETING).
13. [ ] Soumettre à approbation Meta. Délai typique : **quelques heures à 5 jours ouvrés**. Prévoir un lot de templates de secours en cas de rejet (motif fourni par Meta).

### Phase E — Configuration IA (agent Agentia, ~2-3 h)
14. [ ] Rédiger avec le client le **profil d'entreprise** : horaires, tarifs, zone, FAQ, conditions de commande/RDV, consignes de ton.
15. [ ] Générer le **prompt système** + règles de transfert humain (mots-clés : « annuler », « réclamation », « urgent », « parler à quelqu'un » ; compteur de 2 échecs de compréhension ; montants hors fourchette).
16. [ ] Brancher le **transfert humain** : notification push/WhatsApp vers le téléphone du client, reprise de main avec historique complet.
17. [ ] Configurer les **rappels de RDV** (Pro) : envoi de template UTILITY J-1 et H-2, gestion des réponses (confirmer/déplacer/annuler).

### Phase F — Recette et mise en service (agent Agentia + client)
18. [ ] **Tests en bocal** : conversations de test entre deux numéros (collègue ↔ numéro du client) couvrant : question FAQ, demande de RDV, hors-sujet, insulte, demande d'humain, urgence.
19. [ ] Vérifier : envoi des templates, fenêtre 24 h, statuts de message (envoyé/délivré/lu), transferts.
20. [ ] **Formation client (1 h, incluse dans l'installation)** : reprise en main, réglages, tableau de bord, modification des règles.
21. [ ] Annonce de mise en service au client + démarrage de l'essai (14 jours) ou de l'abonnement.

---

## 4. Templates de messages (à créer pour chaque client)

Règle Meta : tout message **hors fenêtre de 24 h** doit utiliser un template approuvé. Variables `{{1}}`, `{{2}}`… entre accolades. Langue `fr`. Catégories :

| Template | Catégorie | Corps (exemple) | Usage |
|---|---|---|---|
| `rappel_rdv_j1` | UTILITY | « Bonjour {{1}}, rappel : votre rendez-vous {{2}} est prévu demain à {{3}}. Répondez CONFIRMER, ou contactez-nous pour le déplacer. » | Rappels de RDV (Pro) |
| `rappel_rdv_h2` | UTILITY | « Bonjour {{1}}, {{2}} commence à {{3}}. Nous vous attendons ! » | Rappel J-0 |
| `confirmation_rdv` | UTILITY | « Bonjour {{1}}, votre rendez-vous du {{2}} à {{3}} est confirmé. Merci ! » | Confirmation après réservation hors fenêtre |
| `confirmation_commande` | UTILITY | « Bonjour {{1}}, votre commande {{2}} est bien enregistrée. Total estimé : {{3}}. Nous revenons vers vous pour la livraison. » | Prise de commande (Pro) |
| `relance_devis` | MARKETING | « Bonjour {{1}}, vous avez demandé un devis {{2}} le {{3}}. Souhaitez-vous que nous le finalisions ? » | Relance (optionnel) |

Pitfalls Meta :
- Pas d'emojis à profusion, pas de promesses de gain (« gagnez 100 € »), pas de mentions concurrentes — motifs de rejet fréquents.
- Les templates MARKETING sont plus chers et soumis à des règles de fréquence plus strictes : privilégier UTILITY dès que le message est transactionnel.
- Toujours prévoir **2-3 variantes** par usage pour absorber un rejet sans bloquer la mise en service.

---

## 5. Coûts (à jour à la date du module — vérifier sur `business.whatsapp.com/products/platform-pricing` avant chaque devis)

Tarifs **par conversation** (une conversation = une fenêtre de 24 h ouverte par un événement) — ordres de grandeur France :

| Type de conversation | Coût indicatif (France) |
|---|---|
| Service (initiée par le client, réponses libres 24 h) | ~0,03 à 0,10 € |
| Utilitaire (rappel RDV, confirmation — templates UTILITY) | ~0,03 à 0,04 € |
| Authentification | ~0,03 à 0,04 € |
| Marketing (templates MARKETING) | ~0,09 à 0,10 € |

**Exemple de calcul pour une TPE type** (promis sur la landing) :
- 350 conversations service/mois × ~0,05 € ≈ **17 €**
- 60 rappels RDV/mois × ~0,03 € ≈ **2 €**
- Total ≈ **15 à 40 €/mois**, réglé directement à Meta par le client (carte ou virement Meta), en sus de l'abonnement Agentia.

Règles internes :
- **Toujours** citer ces frais au client AVANT signature (ils figurent sur la landing, section « Avant de vous lancer »).
- Les tarifs Meta sont **révisables** : re-vérifier avant chaque devis ; un client à fort volume (> 2 000 conversations/mois) mérite une simulation de coût dédiée.
- Aucun frais BSP : branchement direct Cloud API.

---

## 6. Honnêteté commerciale (non-négociable)

Ces points sont **affichés sur la landing** et doivent être répétés au téléphone (cf. `script-pilotage.txt`) :
1. **Validation Meta Business obligatoire** : sans elle, pas de mise en service. Le client fournit les documents, on s'occupe du reste.
2. **Frais Meta en sus** de l'abonnement, facturés par Meta (~0,03–0,10 €/conversation).
3. **Délai de mise en service de 1 à 2 semaines** (validation + approbation templates) : jamais de promesse « prêt demain ».

---

## 7. Checklist de mise en service (résumé — à cocher avant de déclarer un client « live »)

- [ ] Compte Meta Business créé et **entreprise vérifiée** (dossier complet).
- [ ] Numéro dédié enregistré dans le WABA, nom d'affichage validé.
- [ ] Webhook Agentia actif (réception + statuts), token permanent sécurisé.
- [ ] Templates approuvés (rappel J-1, H-2, confirmation RDV, confirmation commande).
- [ ] Prompt IA relu et validé **par le client** (horaires, tarifs, FAQ exacts).
- [ ] Règles de transfert humain testées (mots-clés + compteur d'échecs).
- [ ] Rappels RDV testés en conditions réelles (si Pro).
- [ ] Formation 1 h effectuée ; le client sait reprendre la main et modifier ses règles.
- [ ] Le client a été informé **par écrit** des frais Meta et du délai initial.
- [ ] Essai de 14 jours démarré (ou abonnement actif) ; rappel programmé J+13 pour la bascule.

---

## 8. Points de vigilance opérationnels

- **Fenêtre 24 h** : toute réponse libre doit partir dans la fenêtre. Si l'IA a besoin de plus (devis long), elle envoie un template UTILITY (ouvre une nouvelle fenêtre payante) — à réserver aux cas utiles.
- **Numéro jamais utilisé sur l'app WhatsApp** : cause n°1 des blocages d'enregistrement WABA.
- **Transfert humain** : le téléphone de l'artisan doit avoir l'app **WhatsApp Business** (gratuite) sur le même numéro pour reprendre les conversations — vérifier lors de l'installation.
- **Multi-employés** (Pro) : un seul numéro = un seul agent WhatsApp. Pour plusieurs personnes, prévoir un partage de compte ou une file de transfert (cas à chiffrer séparément).
- **RGPD** : clause DPA fournie au client (traitement des messages par l'IA) ; pas de conservation des messages au-delà de la réponse ; pas d'utilisation publicitaire des données.
- **Facturation Agentia** : abonnement mensuel (virement ou prélèvement), installation 199 € facturée à la bascule payante ; frais Meta jamais prélevés par Agentia.

---

## 9. Liens utiles

- Console développeur : `developers.facebook.com` (app + produit WhatsApp, webhook)
- Espace Business : `business.facebook.com` (WABA, vérification, templates)
- Tarifs officiels : `business.whatsapp.com/products/platform-pricing`
- Docs Cloud API : `developers.facebook.com/docs/whatsapp/cloud-api`
- Compte de contact Agentia : `agentiadeploiement@gmail.com`
