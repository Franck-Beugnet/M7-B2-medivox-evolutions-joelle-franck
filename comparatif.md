# Comparatif des 3 options — 4 dimensions

> **Prédicteur de séjour prolongé (DMS) — MediVox**  
> Arbitrage d'architecture : A (ML classique modernisé) vs B (Hybride LLM extraction → ML) vs C (Multi-agents).  
> Seules la **Sobriété** et la **Performance** sont chiffrées en ordres de grandeur. La **Conformité** et l'**Évolutivité** sont qualifiées par un niveau justifié, avec 2 risques majeurs et leurs mesures de maîtrise. Chaque chiffre renvoie formellement à une hypothèse de référence ($H_1$ à $H_6$).

---

## Matrice comparative 3 × 4

| Dimension | A — ML classique modernisé | B — Hybride LLM extraction → ML | C — Multi-agents (LangGraph) |
|---|---|---|---|
| **Sobriété** *(chiffrée)* | **~50 € / mois** (~0,005 € / dossier)<br/>• Énergie : **~15 kWh / mois** (CPU partagé)<br/>*(Réf. H1, H3a)*<br/>⚠️ *Risque coût : sous-estimation de la maintenance MLOps si dérive fréquente.* | **~250 € à 400 € / mois** (~0,025 à 0,040 € / dossier)<br/>• API LLM : ~33 €/mois + stack HDS/GPU : ~200-350 €/mois<br/>• Énergie : **~80 à 120 kWh / mois**<br/>*(Réf. H1, H2, H3b)*<br/>⚠️ *Risque coût : explosion de la facture si le volume de texte ou la taille des CR double.* | **~1 200 € à 1 800 € / mois** (~0,12 à 0,18 € / dossier)<br/>• 4 à 5 appels LLM/dossier + supervision + GPU dédié<br/>• Énergie : **~400 à 600 kWh / mois**<br/>*(Réf. H1, H2, H3c)*<br/>⚠️ *Risque coût : dérive budgétaire incontrôlée par boucles d'agents et réitérations.* |
| **Performance** *(chiffrée)* | **F1 = 0,72** (classe séjour prolongé)<br/>• Abstention (rejet) : **~12 %** (zone [0,40 ; 0,65])<br/>• Latence p95 : **< 50 ms**<br/>*(Réf. H1, H4, H6)*<br/>⚠️ *Risque perf : plafond de verre sur les cas où le risque dépend d'un signal textuel non codé.* | **F1 = 0,76 à 0,78** (*gain conditionné à H5*)<br/>• Abstention : **~8 %** (meilleure séparation)<br/>• Latence p95 : **~2 500 ms** (extraction LLM)<br/>*(Réf. H1, H2, H4, H5)*<br/>⚠️ *Risque perf : perte de gain si le taux d'erreur d'extraction LLM dépasse 10 %.* | **F1 = 0,70 à 0,73** (instabilité des prompts)<br/>• Abstention : **~18 %** (désaccords inter-agents)<br/>• Latence p95 : **~12 000 à 15 000 ms**<br/>*(Réf. H1, H2, H3c)*<br/>⚠️ *Risque perf : variabilité stochastique dégradant la calibration du risque et l'adhésion soignante.* |
| **Conformité** *(qualifiée)* | **Niveau : FORT**<br/>• **Risque 1** : Décision automatisée sans traçabilité (RGPD art. 22).<br/>*Maîtrise* : Seuil d'incertitude avec routage humain (HITL) + logs explicatifs SHAP conservés 5 ans.<br/>• **Risque 2** : Fuite de données sensibles.<br/>*Maîtrise* : Données cantonnées au réseau interne hospitalier (On-Premise / HDS), zéro transfert externe. | **Niveau : INTERMÉDIAIRE**<br/>• **Risque 1** : Transfert de données de santé hors UE / non-HDS (RGPD art. 9).<br/>*Maîtrise* : Utilisation stricte d'un modèle souverain certifié HDS ou auto-hébergé localement.<br/>• **Risque 2** : Hallucination de variables cliniques impactant la décision.<br/>*Maîtrise* : Schema JSON strict (Structured Outputs), contrainte de citation textuelle, imputation neutre (`null`) si doute. | **Niveau : FAIBLE**<br/>• **Risque 1** : Boîte noire multi-niveaux et perte de traçabilité (AI Act art. 13 & 14).<br/>*Maîtrise* : Journalisation intégrale de l'état partagé (*State Store*) et des prompts/réponses de chaque agent.<br/>• **Risque 2** : Dérive d'autonomie et validation humaine fictive (biais d'automatisation).<br/>*Maîtrise* : Procédure de double signature médicale obligatoire en cas d'escalade. |
| **Évolutivité** *(qualifiée)* | **Niveau : INTERMÉDIAIRE**<br/>• Nouveaux flux : simple pour tables structurées (biologie, constante) ; nul pour le texte.<br/>• Échelle : supporte 100 000 dossiers/mois sans refonte.<br/>⚠️ *Point de rupture : incapacité structurelle à intégrer la parole ou les notes cliniques libres.* | **Niveau : FORT**<br/>• Nouveaux flux : ajout aisé de nouvelles variables extraites par enrichissement du prompt/schéma JSON.<br/>• Échelle : horizontalement scalable (batching d'extraction asynchrone).<br/>⚠️ *Point de rupture : saturation de la fenêtre de contexte si agrégation de dossiers multi-séjours.* | **Niveau : FAIBLE**<br/>• Nouveaux flux : nécessite d'ajouter de nouveaux agents et de rééquilibrer le graphe complet.<br/>• Échelle : scaling très coûteux en GPU et bande passante.<br/>⚠️ *Point de rupture : explosion combinatoire des états du graphe et effets de bord imprévisibles.* |

> **Définition commune de l'Évolutivité** : Capacité du système à intégrer de **nouvelles sources de données** (texte, biologie temps réel, signaux), à faire **évoluer le modèle** (ré-entraînement, ajout de variables) et à **augmenter le volume** (charge d'admission) sans refonte logicielle majeure.

---

## Table des Hypothèses Chiffrées (H1 à H6)

Tout chiffre du tableau renvoie strictement aux hypothèses explicites ci-dessous (ordres de grandeur honnêtes d'ingénierie, pas des prédictions budgétaires au centime) :

| # | Hypothèse | Valeur retenue | Source / Méthode de calcul / Date |
|---|---|---|---|
| **H1** | **Volume d'activité mensuel** | **10 000 dossiers / mois** | Établissement pivot ou GHT (~350 à 500 séjours analysés par jour ouvré). |
| **H2** | **Volumétrie textuelle d'un compte-rendu** | **1 200 tokens in** (CR urgences/admission)<br/>**150 tokens out** (JSON structuré) | Constat moyen sur les comptes-rendus hospitaliers français (~800 à 1 000 mots). Payload JSON : 4-6 variables typées + citation source. |
| **H3a** | **Tarif hébergement Option A (ML CPU)** | **~50 € / mois** | VM Linux standard 2 vCPU / 8 Go RAM mutualisée sur infrastructure hospitalière existante. Compute < 10 ms par dossier. |
| **H3b** | **Tarif API LLM souverain HDS Option B** | • Input : **0,20 € / M tokens**<br/>• Output : **0,60 € / M tokens**<br/>*Total API* : ~33 € / mois<br/>*Total avec VM HDS/API managée* : **~250-400 € / mois** | Calcul : $10\,000 \times (1\,200 \times 0{,}20 \cdot 10^{-6} + 150 \times 0{,}60 \cdot 10^{-6}) = 10\,000 \times (0{,}00024 + 0{,}00009) = 3{,}30$ €/mois si batch brut, majoré à **~33 €/mois** avec rejeu/contrôle prompt, complété par l'abonnement/instance gateway sécurisée HDS (~200 à 350 €/mois) ou instance GPU L4 (~350 €/mois). |
| **H3c** | **Tarif Multi-agents Option C** | **~4 500 tokens in / 800 tokens out par dossier**<br/>(4 à 5 appels LLM chaînés)<br/>*Total infra* : **~1 200 à 1 800 € / mois** | Calcul tokens : $10\,000 \times (4\,500 \times 0{,}20 \cdot 10^{-6} + 800 \times 0{,}60 \cdot 10^{-6}) \approx 138$ €/mois en API pure. Nécessite une instance GPU dédiée haute disponibilité pour absorber la latence (ex. 2x NVIDIA A10G/L4 HDS avec vLLM) $\approx$ 1 200 à 1 500 €/mois fixe. |
| **H4** | **Baseline de performance M7-B1** | **F1 = 0,68** sur la classe séjour prolongé<br/>Latence p95 = 45 ms | Audit M7-B1 : Modèle tabulaire initial non monitoré, déséquilibre de classe (~20 % de séjours prolongés > 7 jours), sans seuil d'abstention. |
| **H5** | **Protocole d'ablation obligatoire** | Validation formelle exigée sur test set holdout 20 % | Aucun gain de l'Option B ($F1 = 0{,}76-0{,}78$) n'est considéré comme acquis sans preuve par ablation : même modèle tabulaire entraîné et testé *avec* et *sans* les features extraites par LLM, sur le même jeu de données. |
| **H6** | **Impact du seuil de rejet sur Option A** | Gain de F1 : **0,68 → 0,72**<br/>Taux de rejet : **12 %** | En rejetant les scores ambigus ($0{,}40 \le P < 0{,}65$) vers la revue humaine, le modèle automatisé élimine la majorité des faux positifs/négatifs marginaux. |

---

## Synthèse comparative des risques

1. **Option A (ML modernisé)** : Le risque n'est pas réglementaire ni financier, il est **métier** : le système reste aveugle aux détresses sociales ou pertes d'autonomie uniquement consignées par écrit dans le dossier.
2. **Option B (Hybride)** : Le risque est la **qualité de l'extraction** : si le LLM extrait des informations fausses et que la validation échoue, le modèle ML prendra de mauvaises décisions sur des données hallucinées. D'où l'importance de l'imputation par `null` et de l'audit soignant (échantillonnage 5 %).
3. **Option C (Multi-agents)** : Le risque est le **sur-engineering total** : une infrastructure disproportionnée, un coût décuplé, une latence inadaptée aux urgences et un non-déterminisme difficilement défendable face à l'AI Act.
