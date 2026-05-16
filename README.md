# KookiiClean — Bot WhatsApp Devis & RDV

Bot WhatsApp automatisé pour Kooki Clean (lavage automobile à domicile).  
Gère les devis, prises de RDV, photos véhicule, et confirmation calendrier.

---

## IDs importants (serveur n8n EasyPanel)

| Élément | ID |
|---|---|
| Workflow principal (WhatsApp V2) | `21VOX7IzmTTPK11v` |
| Debounce Store (workflow vide isolé) | `WGAFZ4dHqrCt2K5Z` |
| Credential Google Drive | `vgYK30iGTlarOvOj` |
| Credential Google Calendar | `h0JBzDodIfM1PB8r` |
| Credential OpenAI | `yKzgSw6lPM36G7fm` |
| Instance Evolution API | `KookiiClean` |

Serveur n8n : `novaflows-core-n8n.cvl1v8.easypanel.host`  
Config complète : `config.js` (ne pas commit — contient la clé API)

---

## Architecture du workflow

WhatsApp Trigger (webhook Evolution API)
→ Message Privé? (filtre groupes @g.us — ignorés)
→ Debounce Read (GET store isolé)
→ Debounce Init (Code — collecte messages sur 5s)
→ Debounce Write (PUT store isolé)
→ Dernier Message? (IF _proceed)
└─ FALSE → stop
└─ TRUE → Wait 5s
→ Debounce Read 2
→ Debounce Check (fusionne textes / collecte photos)
→ Latest? (IF _isLatest)
└─ FALSE → stop
└─ TRUE → Route: Message ou Callback?



**Branche Message (texte) :**
Extraire Données Message
→ Lire Formules (Google Sheets)
→ Agent IA Kookii Clean (GPT-4o + mémoire)
→ Détecter et Stocker RDV
→ Détecter Phase (devis / booking / annulation)
→ [si RDV complet] Vérif dispo calendrier
→ Préparer Récap + boutons Accepter/Refuser
→ Envoyer au client



**Branche Photo :**
Photo ou Texte? → Détecter Contexte Photo → Photo Booking?
→ Obtenir Base64 Media (Evolution API)
→ Créer Fichier Binaire
→ Upload Drive (root "My Drive")
→ Rendre Photo Publique
→ Stocker URL Drive (dans staticData)
→ Envoyer "Photo reçue" + Préparer Récap



**Branche Callback (boutons) :**
Extraire Données Callback → Accepté ou Refusé?
→ [Accepté] Récupérer Booking + Préparer Cal
→ Créer Événement Calendrier
→ Confirmer RDV Client + Notifier Patron
→ Enregistrer dans CRM Notion
→ [Refusé]  Confirmer Annulation + Enregistrer Refus CRM



---

## Système Debounce (v12)

**Problème résolu** : plusieurs messages envoyés rapidement (ex: "Pierre" puis "Dupont" en 600ms) sont fusionnés en un seul avant d'être envoyés à l'IA.

**Architecture** : stockage dans un workflow n8n vide dédié (DEBOUNCE_STORE) via l'API REST n8n. Ce workflow n'est jamais exécuté → pas de conflit avec les agents IA qui écrivent aussi dans le staticData du workflow principal.

**Script de déploiement** : `scripts/fix_debounce_v12.js`

---

## Credentials n8n nécessaires

| Service | Usage |
|---|---|
| **Evolution API** | Envoyer/recevoir messages WhatsApp |
| **OpenAI** | Agent IA (GPT-4o) + analyse photos (Vision) |
| **Google Drive** | Upload photos véhicule |
| **Google Calendar** | Créer événements RDV |
| **Google Sheets** | Lire les formules/tarifs |
| **Notion** | CRM (bookings, refus, annulations) |

---

## Ce qui fonctionne

- ✅ Conversation IA (devis, questions, objections)
- ✅ Extraction automatique des infos RDV ([BOOKING] JSON)
- ✅ Vérification disponibilité calendrier avant confirmation
- ✅ Boutons Accepter/Refuser (Evolution API interactive messages)
- ✅ Création événement Google Calendar avec détails + photos
- ✅ Analyse photo IA (GPT-4 Vision → état du véhicule → prix)
- ✅ Upload photos sur Google Drive + lien public dans le calendrier
- ✅ Debounce v12 : fusion messages rapides (fenêtre 5s)
- ✅ Filtre groupes WhatsApp (ignorés)
- ✅ CRM Notion (bookings, refus, annulations)
- ✅ Commandes admin : `/reset`, liste RDV, stats owner
- ✅ Gestion annulations (calendrier + CRM)
- ✅ Système devis avec validation patron

---

## En cours / À implémenter

### 🔲 Multi-photos Drive (priorité haute)

**Problème** : quand le client envoie 2 photos rapidement, le debounce bloque la 2ème (`_proceed: false`). Seule la 1ère est uploadée. Le calendrier n'a qu'une photo.

**Solution planifiée** :
1. Stocker `messageData` dans chaque item de la queue debounce (modifier `Debounce Init`)
2. `Debounce Check` retourne toutes les photos en `_photoItems: [data1, data2]`
3. Nouveau nœud `Créer Dossier Drive` — 1 dossier par client (`kookii_33xxx`)
4. Nouveau nœud `Rendre Dossier Public` — partage le dossier
5. Nouveau nœud `SplitOut Photos` — itère sur chaque photo
6. `Upload Drive` → upload dans le dossier au lieu de root
7. `Stocker URL Drive` → stocke `drive.google.com/drive/folders/ID`
8. `Envoyer Photo Reçue` → `executeOnce: true`

Résultat : le calendrier montre un lien galerie avec toutes les photos.

### 🔲 Optimisation trajets (roadmap)

Proposer des créneaux qui minimisent les déplacements entre clients (V3).

---

## Relancer le projet

1. Vérifier que le workflow `21VOX7IzmTTPK11v` est **actif** sur n8n
2. Vérifier que le webhook Evolution API pointe vers `https://novaflows-core-n8n.cvl1v8.easypanel.host/webhook/kookii-whatsapp`
3. Vérifier que les credentials Google sont toujours valides (OAuth2 expire)
4. Pour redéployer le debounce : `node scripts/fix_debounce_v12.js`
