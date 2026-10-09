# Supervision thermique des salles serveurs — Cahier des charges

Structure de la présentation (5 points) : **Contexte → Besoin → Objectifs → Fonctions → Synoptique**

---

## 1. Contexte

- L'établissement dispose de **2 salles serveurs** : une à **Copernic** et une à **Fauriel**.
- Ces salles hébergent des **équipements critiques** : serveurs, équipements réseau, firewall.
- Une **panne de climatisation** peut provoquer une surchauffe des équipements informatiques et perturber les services.
- **Aucune supervision thermique automatisée** n'est en place.

## 2. Besoin

Surveiller en continu la température des salles serveurs et être alerté automatiquement en cas de dépassement, afin d'éviter tout risque de surchauffe du matériel informatique et de garantir la continuité de service.

## 3. Objectifs

- Surveiller la température des deux salles serveurs en continu.
- Alerter automatiquement en cas de température trop élevée.
- Afficher les températures sur un écran dans le bureau informatique.
- Rendre les données consultables sur une **page Web de supervision**.

## 4. Fonctions

Le système devra :

1. **Mesurer** la température à au moins deux endroits dans chaque salle serveur.
2. **Remonter** les données en temps réel sur la page Web de supervision, par **protocole MQTT**.
3. **Déclencher une alerte** si un seuil de température défini est dépassé.
4. **Envoyer un email** en cas de dépassement critique.
5. **Permettre un affichage permanent** des températures dans le bureau informatique.

### Périmètre

**Inclus dans le projet**

- Installation de capteurs de température dans les deux salles serveurs.
- Raccordement des capteurs au réseau et connexion à la page Web de supervision.
- Publication des mesures en **MQTT** vers le broker, abonnement de la page Web.
- Alerte par email en cas de température trop élevée.
- Affichage des températures sur un écran dans le bureau informatique.

**Non inclus dans le projet**

- La réparation ou le remplacement des climatisations.
- La modification des installations électriques ou des équipements existants.
- Les autres locaux du bâtiment.

## 5. Synoptique

> **Évolution de l'architecture :** la supervision n'est **plus assurée par un serveur CheckMK** et **n'utilise plus le protocole SNMP**. La supervision est assurée par une **page Web de supervision** dédiée, alimentée par **MQTT** (broker Mosquitto).

Schéma éditable fourni en fichier Draw.io : **`Synoptique.drawio`**

### 5.1 Chaîne fonctionnelle
```
   SALLE COPERNIC                    SALLE FAURIEL
  ┌────────────────────────────┐    ┌────────────────────────────┐
  │DS18B20 #1  (T1)            │    │DS18B20 #3  (T3)            │
  │DS18B20 #2  (T2)            │    │DS18B20 #4  (T4)            │
  │                            │    │                            │
  │Bus 1-Wire  GPIO17          │    │Bus 1-Wire  GPIO17          │
  │GND / 3V3 / DATA            │    │GND / 3V3 / DATA            │
  │pull-up 4,7 kΩ              │    │pull-up 4,7 kΩ              │
  └────────────────────────────┘    └────────────────────────────┘

                 │                                 │
  ┌────────────────────────────┐    ┌────────────────────────────┐
  │ESP32-S3-ETH #1             │    │ESP32-S3-ETH #2             │
  │Ethernet W5500              │    │Ethernet W5500              │
  │IP fixe 192.168.1.11        │    │IP fixe 192.168.1.12        │
  │Client MQTT                 │    │Client MQTT                 │
  └────────────────────────────┘    └────────────────────────────┘
                 │  Ethernet 10/100 Mbps           │
                 └────────┐               ┘────────└
                          └───────────────┘

                                  ▼
                   ┌──────────────────────────────┐
                   │Switch — VLAN Supervision     │
                   └──────────────────────────────┘
                                  │  Ethernet
                                  ▼
                   ┌──────────────────────────────┐
                   │Firewall                      │
                   │filtrage / cloisonnement VLAN │
                   └──────────────────────────────┘
                                  │  Ethernet
                                  ▼

     ┌────────────────────────────────────────────────────────┐
     │ SERVEUR DE SUPERVISION — IP 192.168.1.10               │
     ├────────────────────────────┬───────────────────────────┤
     │ Broker MQTT                │ Page Web de supervision   │
     │ Mosquitto                  │ HTML + JavaScript         │
     │ TCP 1883 / WebSocket 9001  │ abonnée aux topics        │
     │ QoS 1 — retain             │ temperature/...           │
     ├────────────────────────────┴───────────────────────────┤
     │ Règles de seuil                                        │
     │   T < 27 °C       : Normal                             │
     │   27 ≤ T < 35 °C  : Avertissement                      │
     │   T ≥ 35 °C       : Critique                           │
     │                                                        │
     │ Alerte email               │ Écran bureau              │
     │ Service informatique       │ informatique              │
     └────────────────────────────▲───────────────────────────┘
                                  │   flux MQTT   (PUBLISH / RETAIN / LWT)

              PUBLISH   temperature/<salle>/<capteur>
              RETAIN    dernière valeur conservée par le broker
              LWT       statut/<équipement> = online / offline
```

### 5.2 Détail du nœud de mesure ESP32 (côté capteur)

Câblage du capteur DS18B20 sur une carte ESP32-S3-ETH :

```
  DS18B20                     ESP32-S3-ETH
  ┌────────┐                  ┌──────────┐
  │  VCC   ├──────────────────┤ 3V3      │
  │        │                  │          │
  │  GND   ├──────────────────┤ GND      │
  │        │                  │          │
  │  DQ    ├──────┬───────────┤ GPIO17   │  bus 1-Wire
  │        │      │           └──────────┘
  │        │  ┌───┴───┐
  │        │  │▭▭▭▭▭▭│  4,7 kΩ
  │        │  └───┬───┘  (pull-up vers 3,3 V)
  │        │      │           ┌──────────┐
  │        │      └───────────┤ 3V3      │
  └────────┘                  └──────────┘
```

| Fil | Couleur | Broche ESP32 | Rôle |
|---|---|---|---|
| `3,3 V` | Rouge | 3V3 | Alimentation du capteur |
| `GND` | Noir | GND | Référence de tension |
| `DATA` | Jaune | **GPIO17** | Bus 1-Wire (données + retour) |
| Pull-up | — | 3V3 → GPIO17 | **Résistance 4,7 kΩ** obligatoire |

> Le nœud de mesure ESP32-S3-ETH peut embarquer **1 à 2 capteurs DS18B20** en parallèle sur le même bus 1-Wire (GPIO17 + 4,7 kΩ) ; les capteurs sont alors distingués par leur **adresse 8 bits** (code de famille `0x28`).

### 5.3 Communication MQTT

| Élément | Valeur |
|---|---|
| Protocole applicatif | **MQTT 3.1.1** (TCP 1883) |
| Transport navigateur | **WebSocket** (port 9001) |
| QoS | **1** (au moins une fois) |
| Retain | **oui** (dernière valeur conservée par le broker) |
| Authentification | Identifiant / mot de passe sur le broker |
| Keep-alive | 60 s, **Last Will and Testament** = `online`/`offline` |

**Arborescence des topics :**

| Topic | Éditeur | Abonné(s) | Contenu |
|---|---|---|---|
| `temperature/copernic/t1` | ESP32 #1 | Page Web | Température T1 en °C (JSON) |
| `temperature/copernic/t2` | ESP32 #1 | Page Web | Température T2 en °C (JSON) |
| `temperature/fauriel/t3` | ESP32 #2 | Page Web | Température T3 en °C (JSON) |
| `temperature/fauriel/t4` | ESP32 #2 | Page Web | Température T4 en °C (JSON) |
| `alerte/temperature` | Page Web | Service informatique | Message d'alerte formaté |
| `statut/<equipement>` | ESP32 #1 / #2 | Page Web | `online` / `offline` |

> Le dernier point de mesure connu est conservé en mémoire par le broker grâce au drapeau **retain** : la page Web affiche une valeur même si un ESP32 est momentanément hors ligne.

### 5.4 Légende / chaîne fonctionnelle

| Étape | Élément | Rôle |
|---|---|---|
| 1 | Capteurs DS18B20 (1 à 2 par salle) | Mesure de la température |
| 2 | ESP32-S3-ETH (1 par salle) | Lecture 1-Wire, mise en forme JSON, publication MQTT |
| 3 | Switch — VLAN Supervision | Transport des données Ethernet |
| 4 | Firewall | Filtrage et cloisonnement du VLAN Supervision |
| 5 | Broker MQTT Mosquitto | Réception, conservation (retain) et distribution des messages |
| 6 | Page Web de supervision | Affichage, application des règles de seuil, alarmes |
| 7 | Notification email | Alerte au service informatique |
| 8 | Écran bureau informatique | Affichage permanent |

### 5.5 Adressage IP

| Équipement | Adresse IP | VLAN |
|---|---|---|
| ESP32-S3-ETH #1 (Copernic) | 192.168.1.11 | Supervision |
| ESP32-S3-ETH #2 (Fauriel) | 192.168.1.12 | Supervision |
| Serveur de supervision (Mosquitto + page Web) | 192.168.1.10 | Supervision |
| Poste du bureau informatique | Via DHCP | Bureau |

### 5.6 Seuils de température : *(à valider)*

| Niveau | Température | Action |
|---|---|---|
| Normal | T < 27 °C | Aucune |
| Avertissement | 27 °C ≤ T < 35 °C | Alerte visuelle dans la page Web |
| Critique | T ≥ 35 °C | Alerte visuelle dans la page Web + email au service informatique |

> Les règles de seuil sont appliquées **côté page Web** (consumer MQTT), à réception de chaque message publié sur les topics `temperature/...`.
