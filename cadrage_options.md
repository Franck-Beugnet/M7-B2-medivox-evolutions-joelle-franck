# Cadrage des 3 options architecturales — Prédicteur DMS MediVox

> **Livrable Phase 1** (Tâche macro 1 — Cadrage & Hypothèses)  
> **Contexte** : Suite à l'audit M7-B1, arbitrage objectif demandé par Hélène Tournier sur l'évolution du prédicteur de « séjour prolongé ».  
> **Auteurs** : Binôme Franck & Joëlle (Atos / MediVox)

---

## 1. Clarification préalable : Pourquoi pas de RAG ici ? (En 3 points)

> ⚠️ **Réponse à l'équipe d'Hélène Tournier** : Le RAG n'est pas retenu pour le prédicteur DMS, et ce n'est pas un oubli :
>
> 1. **Un RAG répond, il ne prédit pas** : Un RAG extrait des textes pour générer une réponse en langage naturel. Il n'a ni probabilité calibrée, ni seuil de décision, ni métrique de classification (F1-score).
> 2. **Sur-engineering inutile** : Le compte-rendu d'admission (~1 200 tokens) tient directement dans le contexte d'un LLM. Créer une base vectorielle et des embeddings pour découper 2 pages serait un surcoût financier et technique gratuit.
> 3. **Deux produits distincts** :
>    - **Prédicteur DMS (notre périmètre)** : Données patient $\rightarrow$ Score de probabilité $\rightarrow$ Architecture A, B ou C.
>    - **Assistant documentaire soignant (hors périmètre)** : Question médecin $\rightarrow$ Synthèse textuelle sourcée. C'est un produit utile, mais distinct : on ne le compare pas au prédicteur.
>
> Pour exploiter le texte dans la prédiction, l'architecture rigoureuse est l'**Option B (extraction LLM JSON $\rightarrow$ modèle tabulaire ML)**.

---

## 2. Description détaillée des 3 options

### Option A — ML classique modernisé (Baseline industrielle optimisée)

* **Principe** :
  L'architecture actuelle (modèle tabulaire de classification binaire type XGBoost / LightGBM) est conservée et industrialisée. Elle n'exploite que les données structurées issues du Dossier Patient Informatisé (DPI) et du PMSI (âge, sexe, mode d'entrée, service d'admission, antécédents codés CIM-10, constantes vitales initiales, actes CCAM). La modernisation porte sur :
  1. L'enrichissement du feature engineering tabulaire (ratios temporels, scores de comorbidité de Charlson/Elixhauser automatisés).
  2. L'intégration d'un seuil de rejet/abstention pour les prédictions incertaines.
  3. L'explicabilité locale systématique par valeurs de SHAP.
  4. L'automatisation du cycle MLOps (CI/CD, monitoring de drift des données et des performances avec Evidently).

* **Composants clés** :
  Pipeline d'ingestion ETL tabulaire $\rightarrow$ Pipeline scikit-learn/XGBoost $\rightarrow$ Moteur d'inférence CPU $\rightarrow$ Service d'explicabilité SHAP $\rightarrow$ Base de logs d'inférence.

* **Force majeure** :
  **Sobriété et maîtrise opérationnelle maximales** : coût compute quasi nul (~quelques dizaines d'euros/mois sur CPU existant), latence ultra-faible ($p95 < 50$ ms), conformité RGPD/HDS immédiate (aucune donnée transmise à un tiers, pas de GPU requis) et explicabilité mathématique exacte par feature.

* **Faiblesse majeure** :
  **Cécité textuelle totale** : le modèle ignore 100 % du texte libre des comptes-rendus d'admission et d'urgences, passant à côté de signaux faibles critiques non codifiés (isolement social, perte d'autonomie à la marche, aidants épuisés, observance thérapeutique précaire).

* **Fallback en conception** :
  **Seuil de rejet sur zone d'incertitude** : si la probabilité calibrée $P(\text{séjour prolongé}) \in [0{,}40 ; 0{,}65]$, le système s'abstient de trancher automatiquement et route le dossier avec ses facteurs SHAP vers la cellule de gestion des lits (revue humaine sous 4 h).

---

### Option B — Hybride Extraction LLM $\rightarrow$ ML prédictif (Synergie texte-tabulaire)

* **Principe** :
  Pipeline à deux étages séquentiels :
  1. **Étage 1 (Extraction textuelle contrôlée)** : Dès l'arrivée d'un compte-rendu médical d'urgence ou d'admission, un LLM souverain/HDS analyse le texte libre et en extrait un schéma JSON strictement typé (JSON Schema via *Structured Outputs*). Ce schéma capture des variables ciblées non présentes dans les bases structurées : score d'autonomie estimé (GIR présumé 1 à 6), isolement social avéré (booléen), trouble cognitif aigu (booléen), dénutrition suspectée (booléen), et citation de la phrase source justificative.
  2. **Étage 2 (Validation & Fusion tabulaire)** : Le JSON est validé (conformité du schéma, plausibilité des valeurs). S'il est valide, ces variables discrètes enrichissent le vecteur de features du patient.
  3. **Étage 3 (Prédiction ML)** : Un modèle de Gradient Boosting tabulaire (entraîné sur features DPI + variables extraites) calcule la probabilité de séjour prolongé.

* **Composants clés** :
  Connecteur DPI texte $\rightarrow$ Module de prompt & extraction LLM sous contrainte de schéma JSON $\rightarrow$ Validateur de schéma (Pydantic) $\rightarrow$ Matrice de features enrichie $\rightarrow$ Modèle de classification tabulaire $\rightarrow$ Table de traçabilité d'extraction (texte source / variables extraites / version LLM).

* **Force majeure** :
  **Capture des déterminants clinico-sociaux non structurés** tout en conservant un **moteur décisionnel prédictif calibré, auditable et explicable** (le LLM n'émet aucun diagnostic ni prédiction, il agit uniquement comme un parseur sémantique d'information).

* **Faiblesse majeure** :
  **Complexité d'infrastructure et coût récurrent** : nécessite une stack LLM certifiée HDS (API souveraine ou GPU dédié type L4), ajoute une latence de 1 à 3 secondes par dossier, et introduit un risque d'erreurs d'extraction ou d'hallucinations qui exige un monitoring qualité continu et une file de relecture.

* **Fallback en conception** :
  **Imputation neutre (`null` / médiane) + alerte qualité** : si l'extraction échoue (timeout, format JSON non respecté, confiance basse du LLM), la variable est imputée à `null` (ou valeur médiane de la population) afin de ne jamais bloquer la chaîne de prédiction ML ; le dossier est parallèlement envoyé dans une file d'audit qualité différée pour les soignants/TIM.

---

### Option C — Orchestration Multi-Agents (Réseau d'agents cliniques coopératifs)

* **Principe** :
  Système multi-agents orchestré sous forme de graphe d'états (type LangGraph) où des agents LLM spécialisés collaborent en partageant un état commun (*shared state*) :
  1. *Agent Ingestion & Qualité* : collecte le dossier, anonymise/vérifie la complétude des données administratives et textuelles.
  2. *Agent Évaluateur Clinique & Comorbidités* : analyse les antécédents, les constantes et la biologie pour lister les risques somatiques.
  3. *Agent Évaluateur Psycho-Social* : analyse les conditions de vie, l'aidance et l'autonomie pour évaluer le risque de rupture de filière d'aval (EHPAD, SSR).
  4. *Agent Prédicteur/Synthèse* : consolide les évaluations des agents précédents, consulte une base de règles institutionnelles et génère un rapport argumenté avec estimation de la durée de séjour.
  5. *Agent Superviseur / Contrôleur de cohérence* : vérifie la non-contradiction des conclusions et arbitre le routage (décision validée vs déclenchement d'une revue humaine).

* **Composants clés** :
  Moteur d'orchestration de graphe (LangGraph) $\rightarrow$ Registre d'outils et prompts spécialisés $\rightarrow$ Gestionnaire d'état partagé (*State Store*) $\rightarrow$ Module de supervision/garde-fous (*Guardrails*) $\rightarrow$ Interface Human-in-the-Loop (HITL) dédiée.

* **Force majeure** :
  **Modularité extrême et granularité du raisonnement** : capacité à générer une synthèse narrative complète pour l'équipe médicale, explications textuelles multi-angles, adaptation aisée à des cas cliniques atypiques ou complexes.

* **Faiblesse majeure** :
  **Sur-engineering critique et non-viabilité économique** pour la tâche demandée : multiplication par 4 ou 5 des appels LLM par dossier (coût financier et énergétique disproportionné), latence cumulative prohibitive ($p95 > 10$ à 15 s), non-déterminisme décisionnel fort, et complexité d'observabilité et de débogage en environnement HDS.

* **Fallback en conception** :
  **Interruption de boucle / Heuristique de repli + HITL obligatoire** : si le graphe dépasse 4 itérations, si une divergence de score $> 30\,\%$ apparaît entre l'agent clinique et l'agent social, ou en cas d'échec d'un agent intermédiaire, le système interrompt l'orchestration, produit un score d'urgence par heuristique tabulaire simple, et notifie le médecin coordonnateur pour arbitrage manuel avec l'historique complet des échanges d'agents.

---

## 3. Table des Hypothèses Chiffrées de Référence (H1 à H5)

Afin de garantir une comparabilité stricte entre les 3 architectures, tous les calculs de coûts et de performance reposent sur le jeu d'hypothèses unifié suivant :

| # | Hypothèse | Valeur retenue | Justification / Source / Date |
|---|---|---|---|
| **H1** | **Volume d'activité** | **$10\,000$ dossiers / mois** | Établissement de santé moyen/grand (~350 à 500 admissions/jour ouvré, soit ~120 000 séjours/an). Base dimensionnante standard pour un CHU/GHT. |
| **H2** | **Volumétrie textuelle par dossier** | **$1\,200$ tokens in / $150$ tokens out** | Compte-rendu d'admission / passage aux urgences : 800 à 1 000 mots (~1 200 tokens). Extraction JSON structurée : ~100 à 150 tokens de payload JSON. |
| **H3a** | **Tarif Compute Option A (ML classique)** | **$50$ € / mois** | Hébergement mutualisé sur VM Linux standard 2 vCPU / 8 Go RAM existante. Consommation CPU négligeable par inférence (< 10 ms de calcul). |
| **H3b** | **Tarif API LLM HDS Option B (Hybride)** | **$0{,}20$ € / M tokens in**<br/>**$0{,}60$ € / M tokens out** | Tarifs publics modèles d'extraction compacts souverains/HDS (ex. Mistral Small / NeMo / GPT-4o-mini HDS, tarif cloud souverain Q1 2026). Alternative GPU dédié L4 : ~350 €/mois. |
| **H3c** | **Tarif API Multi-agents Option C** | **4 appels LLM par dossier**<br/>(cumul : $4\,500$ tokens in / $800$ tokens out) | Orchestration de 4 agents (qualité, clinique, social, synthèse/superviseur) avec réinjection d'historique et de contexte dans l'état partagé. |
| **H4** | **Baseline historique M7-B1 (Existant)** | **F1 = 0,68** (classe séjour prolongé)<br/>Latence $p95 = 45$ ms | Audit M7-B1 : Modèle tabulaire sur données administratives brutes, déséquilibre de classe (~20 % séjours prolongés), dérive non surveillée. |
| **H5** | **Protocole d'ablation obligatoire** | Validation stricte sur jeu de test figé (test set holdout 20 % partitionné par patient/temps) | **Règle absolue** : Aucun gain de performance de l'Option B ne sera validé sans protocole d'ablation comparant *Modèle Tabulaire seul* vs *Modèle Tabulaire + variables extraites par LLM* à données et hyperparamètres comparables. |

---

## 4. Synthèse d'orientation préliminaire

- L'**Option A** est la candidate naturelle de la **sobriété** et de l'efficience opérationnelle.
- L'**Option B** n'est envisageable que si le **protocole d'ablation (H5)** prouve formellement un saut de détection des séjours prolongés non atteignable par les variables administratives.
- L'**Option C** présente tous les symptômes du **sur-engineering technologique** pour une tâche de classification de durée de séjour, et sera documentée pour démontrer son inadaptation au brief d'Hélène Tournier.
