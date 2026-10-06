Spécification Technique et Système : Simulation Opérationnelle Aéroportuaire (LFPO)
Ce document formalise les spécifications techniques, logiques et logicielles nécessaires au développement en C d'une simulation aéroportuaire multi-agents connectée, vectorielle et multi-rôles inspirée de la plateforme de Paris-Orly (LFPO). Il fait office de référentiel d'architecture pour le moteur, le protocole réseau, la physique simplifiée et les arbitrages opérationnels.
1. Vision et Principes Directeurs d'Ingénierie
Langage cible : C (standard C99 ou C11) compilé avec avertissements stricts (-Wall -Wextra -Wpedantic).
Graphismes et Entrées : Raylib pour la fenêtre, le contexte graphique et les entrées ; cimgui (ou rlImGui) pour les instruments opérationnels et la console de débogage.
Direction Artistique : Esthétique vectorielle industrielle CAD/Radar type A-SMGCS (Advanced Surface Movement Guidance and Control System) sur fond sombre (#0B0E14). Aucune texture matricielle complexe : pistes, voies de circulation, cloisons, aéronefs, flux de personnes et véhicules sont tracés par primitives géométriques (segments, polygones filaires, arcs, étiquettes textuelles typées OACI).
Paradigme Mémoire : Zéro allocation dynamique (malloc/free) dans la boucle de mise à jour (tick loop). Toute la mémoire est préallouée au démarrage sous forme de tableaux contigus à taille fixe (flat pools) identifiés par des index entiers stables et des compteurs de génération.
Topologie Réseau : Modèle client-serveur avec autorité absolue du serveur (Authoritative Server). Le serveur calcule l'intégralité des états physiques, logiques et financiers. Les clients envoient des intentions de commande (commands) et reçoivent des instantanés du monde (snapshots) sérialisés à cadence fixe.
2. Architecture Système et Topologie Réseau

                               ┌─────────────────────────────────┐
                               │   SERVEUR AUTORITAIRE (20 Hz)   │
                               │  - État du monde (WorldState)   │
                               │  - Moteurs ATC, Sol, PAX, Éco   │
                               │  - Console de commandes Debug   │
                               └──────────────┬──────────────────┘
                                              │ Sockets non bloquantes (TCP/ENet)
                                              │ Snapshots sérialisés (20 Hz)
                     ┌────────────────────────┼────────────────────────┐
                     │                        │                        │
                     ▼                        ▼                        ▼
       ┌────────────────────────┐┌────────────────────────┐┌────────────────────────┐
       │   CLIENT 1 : TOUR/ATC  ││   CLIENT 2 : INFRA/SOL ││   CLIENT 3 : TERMINAL  │
       │ - Vue globale macro    ││ - Vue moyenne tarmac   ││ - Vue zoomée couloirs  │
       │ - Strips de vol OACI   ││ - Ravitaillements, FOD ││ - Portiques, Duty Free │
       └────────────────────────┘└────────────────────────┘└────────────────────────┘


Boucle de Simulation et Synchronisation
Le serveur cadence le monde à une fréquence fixe de 20 Hz ($\Delta t = 50\text{ ms}$).
Chaque tick incrémente un compteur global 64 bits (tick_id).
Le serveur lit les commandes réseau entrantes, exécute les sous-systèmes dans un ordre déterministe, puis diffuse le paquet d'instantané (SnapshotPacket) à l'ensemble des clients connectés.
Les clients tournent à la fréquence de rafraîchissement du moniteur (60 ou 144 Hz) et interpolent linéairement les positions entre deux instantanés serveur pour garantir un affichage fluide de la caméra et des entités vectorielles.
Continuité de Session et Robustesse Multijoueur
Tout joueur peut changer de rôle à chaud via un paquet CMD_SWITCH_ROLE. L'attribution des rôles ne verrouille pas l'accès au serveur.
En cas de déconnexion d'un joueur, la simulation ne s'arrête pas. Les processus gérés par le rôle vacant passent en exécution automatique sécuritaire :
ATC vacant : Maintien des aéronefs aux derniers points d'arrêt (holding points), mise en circuit d'attente standard des arrivées.
Infra sol vacante : Poursuite des opérations de ravitaillement déjà engagées, file d'attente passive pour les nouvelles demandes.
Terminal vacant : Débit constant sur les filtres de sûreté sans ajustement dynamique d'ouverture de lignes.
Le système supporte un mode observateur/directeur capable de surveiller tous les indicateurs et de forcer des ordres régulateurs transversaux.
3. Répartition Modulaire et Travail Collaboratif à Quatre
Pour éliminer les risques de conflits sur le dépôt de code, le projet est découpé en interfaces scellées dès l'initialisation. Le contrat d'interface commun réside exclusivement dans le dossier common/.



airport_sim/
├── CMakeLists.txt
├── common/
│   ├── protocol.h          # Paquets réseau, codes commandes, enums d'états
│   ├── types.h             # Types géométriques et structures d'entités fixes
│   └── orly_data.h         # Coordonnées vectorielles réelles de LFPO
├── server/
│   ├── main.c              # Point d'entrée serveur et orchestration des ticks
│   ├── srv_network.c       # Abstraction des sockets, sérialisation [Dev 1]
│   ├── atc_engine.c        # Trajectoires, pistes, sillage, alertes [Dev 2]
│   ├── ground_engine.c     # Ravitaillement, bagages, engins, maintenance [Dev 3]
│   ├── pax_terminal.c      # Files d'attente, contrôle sûreté, duty free [Dev 3]
│   ├── economy_engine.c    # Facturation, cours du fuel, pénalités [Dev 1]
│   └── debug_console.c     # Console d'injection d'événements [Dev 1]
└── client/
    ├── main.c              # Point d'entrée client et initialisation Raylib
    ├── clt_network.c       # Réception dé-sérialisation et buffer d'interpolation [Dev 1]
    ├── render_radar.c      # Moteur de rendu vectoriel CAD / LOD / Caméra [Dev 4]
    ├── ui_strips.c         # Strips électroniques ATC et panneaux de contrôle [Dev 4]
    └── ui_debug_overlay.c  # Interface graphique d'inspection live [Dev 4]


Livrables par Développeur
Développeur 1 — Réseau, Cœur Système et Économie :
Implémente le moteur de sockets (TCP non-bloquant ou ENet), le cycle de sérialisation binaire, la boucle de simulation du serveur, la persistance disque, le modèle financier global et l'interpréteur de commandes de débogage en ligne de commande.
Développeur 2 — Moteur Aéronautique, Pistes et Espaces Aériens (ATC) :
Implémente le graphe de navigation des pistes et voies de circulation d'Orly, l'automate d'état des vols, les calculs de sillage vortex, les procédures LVP, les circuits d'attente, les détections de conflit au sol (incursions de piste) et la génération des urgences en vol.
Développeur 3 — Moteur Sol, Logistique Tarmac et Terminal Passagers :
Implémente le circuit de rotation au parking (turnaround), la flotte d'engins terrestres (ravitailleurs, pousseurs, dégivreurs), le réseau physique de tri des bagages, le calcul de flux particulaire des passagers dans le terminal et l'usure de l'infrastructure.
Développeur 4 — Rendu Vectoriel Raylib, Caméra 2D et Interfaces Utilisateur :
Implémente la Camera2D multi-échelles (du plan général métropolitain jusqu'à l'inspection métrique des portes), le système de LOD vectoriel, l'interpolation d'affichage, les interfaces spécialisées par rôle (strips électroniques de vol, synoptiques d'infrastructures, jauges de flux) et l'IHM d'inspection temps réel via cimgui.
4. Modèle Aéronautique et Exploitation Piste (Inspiré d'Orly — LFPO)
Configuration des Pistes et Vecteurs Réels
La simulation reproduit fidèlement la géométrie de Paris-Orly :
Piste 06/24 (3 650 m) : Piste principale long-courrier et gros porteurs.
Piste 07/25 (3 320 m) : Piste ségréguée parallèle/croisée pour moyens porteurs.
Piste 02/20 (1 800 m) : Piste séquentielle secondaire soumise à restrictions strictes de trajectoire et de vent de travers.
La logique bascule entre deux configurations majeures en fonction de la composante de vent transmise par le moteur météo :
Face à l'Ouest (QFU 24 / 25) : Arrivées et départs orientés Ouest (configuration standard majoritaire à Orly).
Face à l'Est (QFU 06 / 07) : Inversion opérationnelle globale dès que le vent arrière dépasse cinq nœuds sur les pistes 24/25.
Turbulences de Sillage et Catégories OACI
Chaque type d'aéronef est classé selon sa masse maximale au décollage (MTOW) :
Light (L) : Jets d'affaires, appareils régionaux (ex. ATR 72, Embraer 190).
Medium (M) : Famille A320, B737.
Heavy (H) : A350, B777, A330.
Le moteur ATC impose un espacement temporel et métrique obligatoire entre deux aéronefs successifs sur le même axe d'approche ou de décollage :
Heavy derrière Heavy : 4 milles nautiques (ou 90 secondes).
Medium derrière Heavy : 5 milles nautiques (ou 120 secondes).
Light derrière Heavy : 6 milles nautiques (ou 180 secondes).
Non-respect par le joueur : Déclenche une alerte de cisaillement critique entraînant une remise de gaz automatique (go-around) ou un accident structurel si l'appareil est en courte finale.
Représentation du Réseau au Sol (Graphe Vectoriel)
Les voies de circulation (taxiways Alpha, Sierra, Romeo, Whiskey...) sont modélisées sous forme d'un graphe orienté :
Nœuds : Intersections, points d'arrêt avant piste (holding points), entrées de postes de stationnement.
Arêtes : Tronçons de roulement avec largeur maximale admise, vitesse limite (ex. 15 à 30 nœuds) et indicateur d'occupation dynamique (is_blocked).
Tout aéronef au sol suit strictement une route composée d'une chaîne d'arêtes validées par l'ATC. Deux aéronefs face-à-face sur la même arête provoquent un blocage opérationnel nécessitant l'intervention d'un tracteur de repoussage sol pour dégager l'axe.
5. Moteur d'Infrastructure, Tarmac et Logistique Sol
Machine à États du Traitement en Porte (Turnaround)
Dès qu'un aéronef s'immobilise sur son bloc de stationnement et coupe ses réacteurs, l'automate de sol prend le relais :



[ARRIVÉE BLOC] ──► Calage roues & Alimentation 400Hz
                      │
                      ├──► Passerelle connectée ──► Débarquement PAX ──┐
                      ├──► Tapis à bagages      ──► Déchargement BAG  ──┤
                      │                                                │
                      ▼                                                ▼
              Nettoyage Cabine ◄───────────────────────────────────────┘
                      │
                      ├──► Ravitaillement Carburant (Hydrant ou Camion)
                      │
                      ├──► Embarquement nouveaux BAG ──┐
                      ├──► Embarquement nouveaux PAX ──┤
                      │                                │
                      ▼                                ▼
              Retrait Passerelle & Équipements ◄───────┘
                      │
                      ▼
        [AUTORISATION PUSHBACK & DÉPART]


Ravitaillement Kérosène : Bornes Oléoduc vs Camions
Postes au Contact (Orly 1, 2, 3, 4) : Équipés de bouches oléoduc enterrées (hydrant system). Le ravitaillement nécessite uniquement un camion serveur d'oléoduc compact qui raccorde la borne sous-terraine aux réservoirs de voilure. Débit élevé, risque de panne logistique faible.
Postes au Large (Remote Stands) : Nécessitent des camions-citernes dédiés qui effectuent des allers-retours entre le parc de stockage centralisé de carburant et l'avion. Vulnérables aux embouteillages de véhicules de service sur les voies périphériques.
Gestion du Dépôt Central : Une réserve globale en mètres cubes approvisionne l'aéroport. Si les cuves descendent sous le seuil critique de 15 %, les débits de distribution sont réduits de 50 %, puis totalement interrompus à 0 %. L'exploitant doit souscrire des commandes de réapprovisionnement par pipeline ou train d'approvisionnement en anticipant la volatilité des cours du kérosène.
Réseau de Convoyage et Traitement des Bagages
Les bagages déchargés ou enregistrés transitent par un graphe sous-terrain de tapis roulants modélisé sous la forme d'un réseau de transit à capacité finie.
Chaque bagage passe par un sas d'inspection radioscopique (EDS - Explosive Detection System).
Défaillances modélisées :
Bourrage mécanique : Bloque une section de convoyeur, nécessitant l'envoi d'une équipe de maintenance.
Erreur d'aiguillage : Envoie un pourcentage de bagages vers un mauvais vol, générant des litiges financiers et des retards au départ.
Bagage douteux : Déclenche un protocole d'arrêt d'urgence du carrousel pour inspection manuelle approfondie.
Usure, Entretien et Déverglaçage
Dégradation des Chaussées : Chaque passage d'aéronef incrémente la fatigue structurelle des pistes et voies de circulation. L'indice d'usure augmente exponentiellement avec la masse de l'appareil. Une piste non entretenue génère des débris (FOD - Foreign Object Debris).
Inspections Piste : L'exploitant doit envoyer un véhicule d'inspection parcourir la piste à intervalles réguliers. L'opération neutralise la piste pendant quatre minutes mais réinitialise le risque de FOD et d'accidents de pneu.
Pôle Dégivrage : Par températures négatives avec précipitations, les aéronefs doivent obligatoirement recevoir un traitement fluide antigivre avant décollage, soit directement en porte, soit sur des aires dédiées situées en tête de piste pour limiter le temps d'attente avant alignement.
6. Moteur Passagers, Terminal et Sûreté
Dynamique des Flux Particulaires
Les passagers ne sont pas calculés individuellement par pathfinding lourd, mais simulés comme des particules contraintes dans des couloirs vectoriels et des files d'attente discrètes (files FIFO) :
Zone Publique : Arrivée par transports terrestres, orientation vers les comptoirs d'enregistrement de la compagnie ou bornes automatiques.
Poste d'Inspection Filtrage (PIF) : Goulet d'étranglement majeur. Chaque ligne ouverte dispose d'une capacité nominale horaire (ex. 120 passagers/heure). Le temps d'attente augmente proportionnellement à la saturation.
Contrôle Frontière (Aubettes PAF) : Obligatoire pour les vols hors espace Schengen. Ralentissement variable en fonction des effectifs policiers alloués.
Zone Commerciale (Duty Free) & Salles d'Embarquement : Les passagers ayant passé les contrôles y séjournent jusqu'à l'appel de leur vol.
Modèle Commercial des Boutiques
Les boutiques Duty Free occupent des surfaces concédées soumises à redevance d'occupation fixe et pourcentage sur le chiffre d'affaires.
La propension d'achat d'un passager dépend directement de son temps résiduel avant embarquement :
Moins de 20 minutes : Vente nulle (course directe vers la porte).
Entre 25 et 60 minutes : Dépense optimale dans les commerces et points de restauration.
Plus de 90 minutes : Saturation des sièges d'attente et décroissance de l'attractivité d'achat.
Arbitrage Retard Vol vs Passagers Manquants
À l'heure théorique de clôture de l'embarquement (Gate Closure) :
Si un petit groupe de passagers (< 5 %) est bloqué au filtre de sûreté : la porte est fermée, leurs bagages enregistrés sont obligatoirement retirés de la soute (impératif de sûreté de rapprochement bagages-passagers), générant un retard sol modéré de 15 minutes.
Si un groupe massif (> 20 %) est bloqué suite à un engorgement général du terminal : le joueur ou la compagnie aérienne doit arbitrer entre maintenir l'avion au sol (accumulation d'indemnités de retard) ou ordonner le départ à vide partiel (pénalités de mécontentement et frais de réacheminement).
Incidents de Sûreté
Bagage Abandonné : Événement rare déclenchant un périmètre de confinement automatique. Toutes les portes situées dans un rayon de 50 mètres sont neutralisées jusqu'à l'intervention des équipes de déminage.
Passager Refoulé : Blocage ponctuel d'une aubette de police aux frontières pour vérification d'identité, réduisant temporairement le débit de la ligne.
7. Système Économique, Contrats et Progression
Balance Financière (CapEx / OpEx)
Flux
Catégorie
Description du calcul
Revenus
Taxes Atterrissage / Décollage
Forfait fixe + coefficient appliqué à la masse maximale au décollage ($MTOW$).
Revenus
Redevance Passagers
Prélèvement fixe par passager ayant embarqué avec succès.
Revenus
Vente de Kérosène
Marge commerciale appliquée sur chaque mètre cube distribué aux aéronefs.
Revenus
Concessions Duty Free
Loyer de base + intéressement aux ventes générées par le flux piéton.
Dépenses
Salaires Opérationnels
Coût récurrent indexé sur le nombre de lignes de sûreté ouvertes et d'engins sol actifs.
Dépenses
Maintenance des Voies
Frais kilométriques fixes d'entretien des pistes, voies de circulation et balisage.
Dépenses
Pénalités Retard
Indemnités versées aux compagnies pour chaque tranche de 15 minutes de retard imputable à l'aéroport.
Dépenses
Amendes Environnement
Pénalité financière sévère en cas de décollage pendant le couvre-feu (23h30 - 06h00) ou d'infraction bruit.

Contrats Compagnies Aériennes
Les contrats définissent des programmes de vols récurrents sur plusieurs jours virtuels.
L'acceptation d'un contrat engage l'aéroport sur un taux de ponctualité minimal garanti (SLA - Service Level Agreement, ex. 85 % des vols partis avec moins de 15 minutes de retard).
La rupture de contrat pour retards excessifs détruit la réputation de l'aéroport et interdit la renégociation de créneaux avec l'alliance aérienne concernée pendant plusieurs cycles.
Arbre d'Évolution et Déblocage
L'extension des capacités aéroportuaires est régie par un registre d'indicateurs débloquant des modules via un masque de bits binaire (unlocked_features_bitflags) :
Niveau Régional : Opérations à vue, aéronefs de classe Light/Medium uniquement, ravitaillement exclusivement par camions-citernes.
Niveau International : Ouverture des terminaux non-Schengen, mise en service des oléoducs sur les portes au contact, activation des procédures LVP et dégivrage.
Hub Majeur : Accueil des gros porteurs (Heavy), tri automatisé des bagages haute cadence, pistes certifiées ILS CAT IIIb, optimisation prédictive des départs.
8. Moteur Météo, Aléas et Arbre des Urgences (Mayday)
Conditions Météorologiques et Impact Exploitation
Direction et Force du Vent : Détermine l'orientation active des pistes. Tout changement de vent imposant une inversion de sens génère un arrêt temporaire des mouvements pendant 10 à 15 minutes pour purger les axes.
Brouillard et Basse Visibilité (LVP - Low Visibility Procedures) :
Activé lorsque la portée visuelle de piste (RVR) descend sous 600 mètres.
Espacement au sol doublé entre tous les aéronefs.
Vitesse de roulage limitée à 10 nœuds.
Interdiction absolue de croiser une piste sans guidage radar de surface certifié.
Neige et Verglas :
Accumulation progressive mesurée en millimètres sur les surfaces de roulement.
Si l'épaisseur dépasse 3 mm, le coefficient de freinage chute drastiquement.
L'exploitant doit déployer des convois de balayeuses/déneigeuses sur les pistes, ce qui neutralise la piste traitée pendant 20 minutes consécutives.
Arbre de Causalité des Urgences
Les situations d'urgence ne surviennent pas comme des événements arbitraires, mais comme la conséquence directe de négligences opérationnelles :



[Attente excessive en vol imposée par l'ATC] ────► [Pénurie Carburant : MAYDAY FUEL]
                                                   - Priorité absolue d'atterrissage
                                                   - Déroutement ou crash si refus

[Absence d'effarouchement / inspection piste] ───► [Ingestion d'oiseaux au décollage]
                                                   - Panne moteur immédiate
                                                   - Atterrissage d'urgence retour immédiat

[Défaut de balayage de piste / Piste usée] ──────► [Éclatement de pneu au roulage/FOD]
                                                   - Blocage d'une voie ou piste
                                                   - Fermeture de piste requise

[Retard dégivrage par temps givrant] ───────────► [Perte de portance / Alerte décrochage]
                                                   - Procédure d'interruption de décollage


9. Outil Développeur et Console d'Injection en Direct
Pour permettre aux quatre développeurs d'inspecter, reproduire et valider les scénarios sans attendre des heures de jeu, un terminal d'administration temps réel est intégré dans le serveur et accessible sur le client via la touche ² (ou F1).



========================= AIRPORT ENGINE DEBUG CONSOLE =========================
[12:04:15][TICK 14205] Clients connectés: 3 | Vols actifs: 18 | RVR: 350m (LVP ON)
> trigger_emergency --type=ENGINE_FIRE --callsign=AFR22B
[OK] Incident ENGINE_FIRE injecté sur AFR22B. Statut vol basculé sur MAYDAY.
> set_weather --wind_dir=060 --wind_speed=25 --snow_rate=12
[OK] Météo mise à jour. Inversion des pistes LFPO ordonnée (QFU 06/07).
> set_cash 5000000
[OK] Trésorerie fixée à 5 000 000 EUR.
================================================================================


Commandes Disponibles
spawn_aircraft : Génère immédiatement un aéronef en approche.
trigger_emergency : Force un incident (FUEL_EMPTY, ENGINE_FIRE, BIRD_STRIKE, GEAR_FAIL).
set_weather : Écrase la météo courante.
fast_forward : Multiplie la cadence de calcul serveur ($x1$, $x2$, $x5$, $x10$) pour tester la robustesse des flux.
inspect_gate : Affiche la structure mémoire complète d'une porte (étape du turnaround, véhicules affectés, volume kérosène restant).
inspect_pax_flow : Exporte les temps d'attente moyens aux filtres de sûreté et les goulots d'étranglement détectés.
10. Persistance et Structure des Données Mémoire
La sauvegarde de session s'effectue par sérialisation binaire directe de l'état du monde (WorldState). Ce procédé garantit une vitesse d'écriture quasi instantanée, une taille de fichier minimale et une reproductibilité exacte sans ambiguïté de conversion de types.
Structures C Fondamentales (Fichier common/types.h)



C
#ifndef TYPES_H
#define TYPES_H

#include 
#include 

#define MAX_FLIGHTS         128
#define MAX_GATES           64
#define MAX_VEHICLES        128
#define MAX_TAXIWAY_NODES   512
#define MAX_TAXIWAY_EDGES   1024
#define MAX_BAGS_IN_TRANSIT 4096

typedef enum {
    WAKE_LIGHT,
    WAKE_MEDIUM,
    WAKE_HEAVY
} WakeCategory;

typedef enum {
    FLIGHT_AIRBORNE_INBOUND,
    FLIGHT_HOLDING,
    FLIGHT_FINAL_APPROACH,
    FLIGHT_ROLLOUT,
    FLIGHT_TAXI_TO_GATE,
    FLIGHT_AT_GATE_TURNAROUND,
    FLIGHT_PUSHBACK,
    FLIGHT_TAXI_TO_RUNWAY,
    FLIGHT_LINEUP_TAKEOFF,
    FLIGHT_DEPARTED,
    FLIGHT_EMERGENCY
} FlightState;

typedef struct {
    uint32_t id;
    char callsign[8];
    char aircraft_type[8];
    WakeCategory wake_cat;
    FlightState state;
    
    // Position et navigation
    float x, y;             // Coordonnées en mètres locaux (LFPO)
    float altitude;        // En pieds (ft)
    float heading;         // Cap en degrés (0 - 359)
    float speed;           // Vitesse en nœuds (kts)
    
    // Assignations
    uint8_t assigned_runway;
    uint8_t assigned_gate;
    uint32_t current_edge_id;
    
    // État opérationnel
    float fuel_remaining_kg;
    float fuel_required_kg;
    uint16_t pax_onboard;
    uint16_t bags_loaded;
    uint16_t bags_total;
    bool is_emergency;
    uint32_t delay_seconds;
} Flight;

typedef struct {
    uint8_t id;
    char name[8];
    float x, y;
    bool has_hydrant;
    bool is_occupied;
    uint32_t current_flight_id;
    uint8_t turnaround_step; // Étape dans la machine à états de sol
    float service_progress; // Pourcentage de complétion de l'étape courante
} Gate;

typedef struct {
    float wind_direction;    // Degrés
    float wind_speed;        // Nœuds
    float visibility_meters; // Portée visuelle
    float temperature_c;     // Degrés Celsius
    float snow_accumulation; // Millimètres sur pistes
    bool lvp_active;         // Procédures Basse Visibilité
} WeatherState;

typedef struct {
    int64_t balance_cents;         // Solde en centimes d'euros
    uint32_t reputation;          // Score 0 - 1000
    uint32_t total_pax_processed;
    float central_fuel_stock_m3;  // Stockage cuves principales
    float current_fuel_spot_price;
    uint32_t unlocked_features;   // Masque de bits pour l'arbre de recherche
} EconomyState;

typedef struct {
    uint64_t tick_id;
    float sim_time_seconds;
    
    WeatherState weather;
    EconomyState economy;
    
    uint16_t flight_count;
    Flight flights[MAX_FLIGHTS];
    
    uint8_t gate_count;
    Gate gates[MAX_GATES];
    
    // Entités réseau additionnelles (véhicules, bagages)
} WorldState;

#endif // TYPES_H


Mécanisme de Sérialisation Disque
La sauvegarde de l'état complet du monde s'opère via un en-tête de validation suivi de l'écriture brute de la structure :



C
#define SAVE_MAGIC 0x4F524C59 // "ORLY"
#define SAVE_VERSION 1

typedef struct {
    uint32_t magic;
    uint32_t version;
    uint64_t timestamp;
} SaveHeader;

bool SaveGameState(const char *filename, const WorldState *world) {
    FILE *f = fopen(filename, "wb");
    if (!f) return false;

    SaveHeader header = {
        .magic = SAVE_MAGIC,
        .version = SAVE_VERSION,
        .timestamp = (uint64_t)time(NULL)
    };

    fwrite(&header, sizeof(SaveHeader), 1, f);
    fwrite(world, sizeof(WorldState), 1, f);
    fclose(f);
    return true;
}


11. Protocole Réseau et Messages Échangés (Fichier common/protocol.h)
Les échanges reposent sur un en-tête commun fixe de 4 octets (type + length) suivi d'une charge utile spécialisée.



C
#ifndef PROTOCOL_H
#define PROTOCOL_H

#include "types.h"

typedef enum {
    MSG_NONE = 0,
    // Client -> Serveur (Commandes)
    CMD_CONNECT,
    CMD_SWITCH_ROLE,
    CMD_ATC_CLEARANCE,       // Autorisation atterrissage/décollage/roulage
    CMD_GROUND_DISPATCH,     // Envoi camion carburant / pushback / dégivrage
    CMD_GATE_ASSIGN,         // Affectation d'une porte à un vol
    CMD_TERMINAL_CONFIG,     // Ouverture/fermeture de lignes de sûreté
    CMD_DEBUG_INJECT,        // Injection d'ordres par la console développeur
    
    // Serveur -> Client (Snapshots & Événements)
    NET_SNAPSHOT_WORLD,      // Diffusion de l'état global complet (20 Hz)
    NET_ALERT_BROADCAST      // Alarme sonore/visuelle (Mayday, conflit piste)
} NetMessageType;

typedef struct {
    uint16_t type;
    uint16_t payload_size;
} NetHeader;

typedef struct {
    uint32_t flight_id;
    uint8_t command_action;  // 0: HOLD, 1: TAXI_TO, 2: LINEUP, 3: CLEARED_TAKEOFF, 4: CLEARED_LAND
    uint32_t target_waypoint_or_gate;
} AtcClearanceCommand;

typedef struct {
    uint32_t flight_id;
    uint8_t service_type;    // 0: FUEL, 1: DEICE, 2: PUSHBACK, 3: BAGGAGE
    bool priority_flag;
} GroundDispatchCommand;

#endif // PROTOCOL_H


Cette architecture découplée, fondée sur des structures C contiguës, un réseau autoritaire et une boucle sans allocations, permet à l'équipe de concevoir indépendamment la simulation, l'affichage et l'équilibrage tout en garantissant des performances constantes sous forte charge opérationnelle.
