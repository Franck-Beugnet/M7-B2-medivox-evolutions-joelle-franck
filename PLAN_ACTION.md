# Plan de cadrage et Feuille de route — M7-B2 (MediVox)

> **Projet** : Évolution architecturale du prédicteur de durée moyenne de séjour (DMS) / séjour prolongé  
> **Équipe** : Binôme Franck & Joëlle  
> **Date de livraison / Freeze** : Lundi soir avant restitution mardi matin M8  
> **Format de restitution** : 15 min à l'oral en duo, directement sur les schémas, sans diapositives.

---

## 🎯 Rappel du Cadrage & Périmètre Métier

1. **Objet du système** : Prédicteur d'aide à la décision pour anticiper le risque de « séjour prolongé » des patients hospitalisés (optimisation de la gestion des lits et orientation médico-sociale précoce).
2. **Clarification sémantique (Point clé client)** :
   - Un **RAG** conversationnel sert à retrouver de l'information pour répondre aux soignants (produit documentaire distinct hors périmètre du prédicteur).
   - L'**Option B** est une **architecture hybride d'extraction structurée (LLM $\rightarrow$ ML)** : le LLM extrait des variables cliniques normalisées (JSON Schema) depuis les comptes-rendus, validées et injectées dans un modèle tabulaire qui réalise la prédiction.
   - L'**Option C** est une **orchestration multi-agents** (découpage modulaire avec supervision et dialogue entre agents).
   - L'**Option A** est la **modernisation de l'existant ML tabulaire** (ré-ingénierie des features, monitoring, CI/CD, seuil de rejet).

---

## 📋 TODO List Opérationnelle

### Phase 1 : Cadrage métier, description des 3 options & Hypothèses chiffrées (≈ 30 min)
- [x] **T1.1** Définir et formaliser la description fonctionnelle des 3 options :
  - **Option A — ML classique modernisé** : Principe (modèle tabulaire XGBoost/LightGBM ré-entraîné, pipeline de features enrichi, monitoring drift/CI-CD), force (sobriété, explicabilité, conformité HDS/RGPD aisée), faiblesse (ignore le texte libre des comptes-rendus).
  - **Option B — Hybride Extraction LLM → ML prédictif** : Principe (un LLM extrait sous JSON Schema strict des variables cliniques du texte libre, validées et injectées dans le modèle tabulaire ML qui prédit), force (valorise les signaux cliniques textuels non codés), faiblesse (coût récurrent par document, latence d'extraction, risque d'hallucination/incertitude).
  - **Option C — Orchestration Multi-Agents** : Principe (graphe d'agents spécialisés : ingestion, extraction, évaluation médico-sociale, superviseur avec état partagé), force (modularité, traçabilité des étapes de raisonnement), faiblesse (sur-engineering manifeste pour une prédiction tabulaire, latence cumulée, coût d'exploitation élevé, surface d'attaque/panne accrue).
  - **Pourquoi pas de RAG ici (3 points courts)** :
    1. Un RAG répond, il ne prédit rien (pas de probabilité ni de seuil).
    2. Sur-engineering : 1 CR (~1 200 tokens) entre en entier dans le LLM, nul besoin de base vectorielle.
    3. Deux produits distincts : prédicteur de lits (périmètre) vs assistant documentaire soignant (hors périmètre). Pour exploiter le texte dans la prédiction, seule l'Option B (extraction LLM structurée $\rightarrow$ ML) est rigoureuse.
- [x] **T1.2** Définir et consigner la volumétrie de référence : $H_1 = 10\,000$ séjours/mois (~500/jour ouvré).
- [x] **T1.3** Poser le dimensionnement textuel : $H_2 = 1\,200$ tokens in / $150$ tokens out JSON par compte-rendu.
- [x] **T1.4** Fixer la grille tarifaire unifiée ($H_3$) : compute ML CPU (~50 €/mois) vs API LLM souveraine/HDS (~0,20 € / M tokens in, ~0,60 € / M tokens out) vs instance GPU dédiée type L4/A10G (~350-500 €/mois).
- [x] **T1.5** Documenter la baseline M7-B1 ($H_4$ : F1 actuel classe séjour prolongé ≈ 0,68, p95 < 100 ms) et le protocole d'ablation obligatoire ($H_5$) pour prouver tout gain de l'option B.

### Phase 2 : Schémas Mermaid comparables & unifiés (≈ 1 h 15)
- [x] **T2.1** Définir la charte graphique Mermaid transverse (mêmes formes, classes CSS, palette hexadécimale, sens `flowchart LR`).
- [x] **T2.2** Concevoir le schéma de l'**Option A** (ML modernisé + seuil de rejet) dans `schemas/option_a.md`.
- [x] **T2.3** Concevoir le schéma de l'**Option B** (Pipeline d'extraction LLM JSON $\rightarrow$ validation $\rightarrow$ modèle tabulaire ML) dans `schemas/option_b.md`.
- [x] **T2.4** Concevoir le schéma de l'**Option C** (Graphe multi-agents : ingestion, extraction, évaluation, superviseur/HITL) dans `schemas/option_c.md`.

### Phase 3 : Matrice comparative 3×4 & Analyse des risques (≈ 1 h 15)
- [x] **T3.1** Chiffrer la **Sobriété** pour chaque option (€/mois, €/dossier, empreinte énergétique indicative kWh).
- [x] **T3.2** Chiffrer la **Performance** attendue (F1-score classe minoritaire « séjour prolongé », abstention, latence p95).
- [x] **T3.3** Qualifier la **Conformité** (RGPD santé / HDS, AI Act, traçabilité) avec 2 risques majeurs + mesures de maîtrise par option.
- [x] **T3.4** Qualifier l'**Évolutivité** (capacité d'ingestion, points de rupture) avec la même définition sur les 3 colonnes.
- [x] **T3.5** Finaliser le tableau comparatif dans `comparatif.md` (renommé depuis `comparatif_TEMPLATE.md`).

### Phase 4 : Conception des Fallback Strategies (≈ 30 min)
- [x] **T4.1** Détailler la procédure de l'Option A : seuil de rejet sur probabilité incertaine ($[0{,}40 ; 0{,}65]$) $\rightarrow$ routage gestionnaire de lits / cadre de santé sous 4 h.
- [x] **T4.2** Détailler la procédure de l'Option B : échec d'extraction ou format JSON invalide $\rightarrow$ imputation par valeur neutre/médiane (non bloquante) + file d'échantillonnage de contrôle.
- [x] **T4.3** Détailler la procédure de l'Option C : boucle infinie / désaccord d'agents / confiance globale basse $\rightarrow$ bascule heuristique dégradée + escalade médecin référent.

### Phase 5 : Note de comparaison & Recommandation tranchée (≈ 1 h 15)
- [x] **T5.1** Rédiger la décision en une phrase en tête de note.
- [x] **T5.2** Rédiger la synthèse exécutive et le détail des 3 options.
- [x] **T5.3** Formuler la **Recommandation unique** étayée par 3 arguments maximum + condition explicite de changement d'avis.
- [x] **T5.4** Définir le **Garde-fou sobriété** en une règle opérationnelle applicable par l'équipe d'ingénierie.
- [x] **T5.5** Bâtir le **Plan de migration en 3 étapes** (Existant $\rightarrow$ Intermédiaire $\rightarrow$ Cible) avec changement, risque et critère formel de passage.
- [x] **T5.6** Finaliser `note_comparaison.md` (renommé depuis `note_comparaison_TEMPLATE.md`, 3 pages max).
