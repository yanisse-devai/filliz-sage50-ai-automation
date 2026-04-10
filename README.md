# Filliz → Sage 50 — Automatisation Comptable IA
![Badge MVP](https://img.shields.io/badge/Status-MVP%20en%20Production-green) ![N8N](https://img.shields.io/badge/Stack-N8N%20%2B%20Claude%20Sonnet%204-blue) ![RGPD](https://img.shields.io/badge/Conformité-RGPD%20%2B%20AI%20Act-orange) ![Version](https://img.shields.io/badge/Version-1.1-lightgrey)

Workflow d'automatisation intelligent qui transforme, vérifie et livre les fichiers de facturation Filliz au format Sage 50 — sans que le pôle comptabilité n'ouvre Excel une seule fois.

---

## 🎯 Problème & Solution

**Problème :** Filliz et Sage 50 ne parlent pas le même langage. Chaque fichier reçu nécessitait une transformation manuelle de 30 à 60 minutes : dates à recomposer, colonnes inutiles, comptes imputés incomplets, codes de paiement à calculer — absorbé seul par la comptable, sans filet de sécurité.  
**Solution IA :** Workflow N8N + Claude Sonnet 4 qui intercepte le fichier dès réception par mail, le transforme en JavaScript, le vérifie par IA sur 7 règles comptables, et notifie le comptable sur Teams avec le fichier prêt à importer.  
**Impact :** -97% temps de traitement (60 min → 2 min), zéro intervention manuelle, vérification IA systématique avant chaque import.

---

## 🚀 Quick Start

**Prérequis N8N Cloud :**
```
- Compte N8N Cloud (v2.14+)
- Credentials Microsoft (Outlook + Teams + OneDrive)
- Compte OpenRouter avec accès Claude Sonnet 4
```

**Variables à configurer :**
```
OPENROUTER_API_KEY   → Clé API OpenRouter
ONEDRIVE_FOLDER_ID   → 01BO2PU7LHJSVPX5GDQZBYZOBHWKSGMFDZ
TEAMS_CHAT_ID        → ID du chat Teams comptable
```

---

## 🛠 Tech Stack

| Composant | Outil | Raison Choisie |
|---|---|---|
| Orchestration | N8N Cloud v2.14+ | No-code, natif Microsoft, triggers Outlook |
| Transformation | JavaScript (Code Node) | Logique déterministe — zéro ambiguïté |
| Vérification IA | Claude Sonnet 4 via OpenRouter | Meilleur Recall sur tâches de vérification structurée |
| Stockage | Microsoft OneDrive | Intégration native Teams + SharePoint |
| Notifications | Microsoft Teams | Stack déjà utilisé par le CFA Médéric |
| Déclencheur | Microsoft Outlook Trigger | Réception automatique sans action utilisateur |
| Format sortie | CSV séparateur ; | Format natif d'import Sage 50 |

---

## ⚙️ Architecture du Workflow

```
Outlook Trigger (réception fichier Filliz)
        ↓
Extract from File (XLSX → JSON)
        ↓
Code JavaScript — Transformation déterministe
  • Nettoyage numéros de pièce (FCT-, -CFAMED)
  • Recomposition dates JJ/MM/AAAA
  • Complétion comptes imputés (tables de correspondance)
  • Calcul codes de paiement (VIR / avoirs AV-)
  • Filtrage lignes vides
        ↓
Code JavaScript — Pré-filtrage IA (8 colonnes essentielles)
        ↓
Aggregate (1 item JSON pour l'IA)
        ↓
Basic LLM Chain (Claude Sonnet 4 — temp. 0.1)
        ↓
        ┌──────────────────────────────────────────┐
   0 anomalie                            Anomalies détectées
        ↓                                          ↓
   Convert to File (CSV;)              Error detected
   Upload OneDrive                     Warning IT Support (Teams)
   Send message Teams + lien
   Confirmation Comptable (24h timeout)
```

---

## 🔍 PM Insights & Arbitrages Clés

**Arbitrage 1 — JavaScript déterministe vs Agent IA pour la transformation**  
La transformation (nettoyage des numéros de pièce, recomposition des dates, tables de correspondance) aurait pu être confiée à un LLM. Écarté : les règles sont 100% déterministes — un compte imputé manquant a toujours la même correction selon la table. Confier ça à un LLM introduit de la variance là où il n'en faut aucune. L'IA est réservée à la vérification post-transformation, où son raisonnement sur des règles comptables apporte de la valeur.

**Arbitrage 2 — 7 règles explicites vs périmètre ouvert pour l'IA**  
Première version du prompt : l'IA vérifiait "tout ce qui semblait anormal". Elle signalait des anomalies inventées (totaux débit/crédit, sessions dupliquées). Solution : borner strictement à 7 règles numérotées, répétées en début ET fin de prompt, avec instruction explicite d'abstention hors périmètre. Contre-intuitif mais efficace : moins de liberté = moins d'hallucination.

**Arbitrage 3 — Recall prioritaire sur Precision**  
En comptabilité, manquer une anomalie (faux négatif) est un incident critique — erreur d'import dans Sage 50. Signaler une anomalie qui n'existe pas (faux positif) coûte 5 minutes de vérification au comptable. La règle d'arbitrage est documentée : faux négatif = Niveau 1 / faux positif = Niveau 3. Température 0.1 + few-shot examples biaisent l'agent vers le Recall.

**Arbitrage 4 — Human-in-the-loop non désactivable**  
Le système aurait pu importer directement dans Sage 50 après validation IA. Refusé catégoriquement : une erreur comptable non détectée en production a des conséquences légales. Le comptable confirme chaque import via Teams. Ce n'est pas une friction — c'est le garde-fou final qui rend le système déployable dans un contexte réglementé.

**Arbitrage 5 — Température 0.1 vs 0**  
Temperature 0 produit des sorties 100% déterministes mais peut bloquer le raisonnement sur des cas limites. 0.1 conserve une légère variance contrôlée, suffisante pour que l'agent "raisonne" sur les avoirs et les frais équipement sans inventer. Couplé au format contraint (Structured Output Parser), le risque de dérive reste négligeable.

---

## ⚠️ Difficultés Rencontrées & Leçons PM

- **Défi 1 : Hallucinations IA persistantes** — L'IA signalait des anomalies inexistantes (totaux débit/crédit, sessions dupliquées). Résolution : température abaissée à 0.1, few-shot examples, pré-filtrage 8 colonnes, périmètre borné à 7 règles. *Leçon : Encadrer l'IA par des règles explicites d'abstention vaut mieux que de réduire la température seule.*
- **Défi 2 : Output parser retournait le schéma JSON** — L'AI Agent évaluait mal le prompt dans le User Message. Résolution : nœud Code séparé pour construire le prompt avant l'agent. *Leçon : Séparer la construction du prompt de l'appel LLM dans N8N.*
- **Défi 3 : Logique avoirs (AV-) inversée** — Les avoirs généraient un double code de paiement APP. Résolution : détection du préfixe AV- pour appliquer VIR uniquement sur la 1ère ligne crédit. *Leçon : Chaque cas métier exige sa propre branche logique.*
- **Défi 4 : Comptes imputés incorrects sur lignes crédit** — Le fichier source contenait des codes 411 sur des lignes crédit. Résolution : table inversée FOURNISSEUR_VERS_COMPTE. *Leçon : Ne jamais faire confiance au fichier source — toujours valider la logique métier ligne par ligne.*

**Key Takeaway :** 70% du temps passé à itérer sur la logique métier JavaScript et l'encadrement du prompt IA. La transformation déterministe est plus complexe que la vérification probabiliste.

---

## 📈 Résultats & Métriques

| Métrique | Avant | Après | Source |
|---|---|---|---|
| Temps de traitement | 30 à 60 min | < 2 min | Mesure terrain |
| Intervention manuelle | Systématique | Zéro avant confirmation | Observation workflow |
| Taux d'erreur d'import Sage 50 | Estimé > 5% | Cible < 1% | Objectif PRD |
| Coût IA par run | — | 0,185 à 0,50 centime | Logs OpenRouter |
| Délai réception → import confirmé | 1 à 2 jours | < 1 heure | Mesure terrain |
| Disponibilité workflow | — | > 98% | Historique N8N |

---

## 🛟 Plan de Maintenance & Disaster Recovery

**Fallback manuel documenté :** En cas de panne N8N ou API Anthropic, Amina dispose du tutoriel Excel antérieur (~25-30 min vs < 2 min avec N8N). Ce tutoriel est maintenu à jour indépendamment du système.

| Scénario | Action immédiate | Délai résolution |
|----------|-----------------|-----------------|
| N8N inaccessible | Fallback manuel + IT contacte support | < 2h |
| API Anthropic en erreur | Traitement sans vérification IA — Amina validée renforcée | < 30 min |
| Format Filliz modifié | IT met à jour le script JS + fallback manuel | < 4h |
| Credentials Microsoft expirés | Renouveler OAuth2 dans N8N | < 1h |
| OneDrive inaccessible | CSV envoyé par mail directement à Amina | < 15 min |

---

## 🧠 Gouvernance IA

- **Périmètre restreint :** 7 règles comptables documentées. Aucune autre vérification autorisée.
- **Température 0.1 :** Vérification quasi-déterministe — variance minimale contrôlée.
- **Few-shot examples :** 3 cas annotés (facture conforme, non conforme, avoir).
- **Human-in-the-loop :** Amina confirme chaque import. Non désactivable.
- **AI Act européen :** Risque minimal — aucune décision autonome, aucune donnée personnelle au sens strict.
- **Arbitrage Recall > Precision :** Faux négatif = Niveau 1. Faux positif = Niveau 3.

---

## 🗺 Roadmap

| Priorité | Évolution | Horizon |
|----------|-----------|---------|
| P1 | Externaliser tables de correspondance vers Google Sheets | Court terme |
| P1 | Contrôle structurel des colonnes attendues | Court terme |
| P2 | Log des décisions Amina (audit trail) | Moyen terme |
| P3 | Constitution du golden dataset annoté (F1-Score réel) | Q3 2026 |
| P3 | Dashboard suivi des imports | Q3 2026 |

---

## 👨‍💼 Compétences PM Démontrées

- **Vision → Exécution :** PRD complet (Universal Idea Model, Build/Buy/Bake, User Stories INVEST, matrice métriques 4 couches, Devil's Advocate) → POC → MVP en production.
- **Arbitrage technique documenté :** JS déterministe vs agent IA, 7 règles vs périmètre ouvert, Recall > Precision — chaque décision argumentée et tracée.
- **Gouvernance IA :** Stratégie anti-hallucination multi-couches, Human-in-the-loop non négociable, Disaster Recovery documenté.
- **Cross-fonc. :** Comptabilité / Automatisation / IA / Conformité RGPD.
- **Outils :** N8N Cloud, OpenRouter, Claude Sonnet 4, Microsoft 365, JavaScript.

---

## 🚧 Limitations Connues

- Les codes *Service public* et *Autres* dans les tables de frais équipement sont incomplets
- Aucun golden dataset annoté — les seuils F1 sont des cibles, pas des mesures validées en production
- Fallback manuel disponible (~25-30 min) en cas de panne API ou N8N

---

## 🤝 Contact

Issues et suggestions bienvenues !  
Portfolio : [github.com/yanisse-kemel] | LinkedIn : [linkedin.com/in/yanisse-kemel] | ✉️ y.kemel@cfamederic.com

*Projet interne CFA Médéric — Yanisse Kemel, Product Manager IA*
