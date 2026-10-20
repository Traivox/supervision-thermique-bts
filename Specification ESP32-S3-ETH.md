# Document de spécification — ESP32-S3-ETH (Waveshare)

**Projet :** Supervision thermique des salles serveurs
**Établissement :** Copernic / Fauriel
**Auteur :** Étudiants
**Date :** 11/02/2026 — V1.1 (architecture MQTT alignée sur le cahier des charges)
**Source :** [docs.waveshare.com/ESP32-S3-ETH](https://docs.waveshare.com/ESP32-S3-ETH)

---

## 1. Identification du produit

| Caractéristique | Valeur |
|---|---|
| Désignation | ESP32-S3-ETH |
| Constructeur | Waveshare |
| SKU (carte seule) | 28972 |
| SKU (kit avec caméra) | 28770 |
| SKU (version PoE) | 28771 |
| SKU (kit PoE + caméra) | 28772 |
| SKU (variante -M) | 34882 → 34885 |
| Type | Carte de développement à microcontrôleur |
| Processeur | ESP32-S3R8 |

> **Note :** la dénomination « ESP32-S3-ETH » désigne le module ESP32-S3 **doté d'une interface Ethernet filaire W5500**. Sans le suffixe `POE`, la carte **ne possède pas d'alimentation PoE** : seule l'interface d'extension PoE est réservée.

---

## 2. Caractéristiques du cœur de calcul

| Paramètre | Valeur |
|---|---|
| Chipset | ESP32-S3R8 |
| Cœur | Xtensa LX7 **dual-core** |
| Fréquence | jusqu'à **240 MHz** |
| SRAM | 512 KB |
| ROM | 384 KB |
| PSRAM | **8 MB** embarquée |
| Flash | **16 MB** (W25Q128) |
| Wi-Fi | 2,4 GHz (802.11 b/g/n) |
| Bluetooth | 5 (LE) |
| Ethernet | 10/100 Mbps, auto-négociation (W5500, interface SPI) |
| Alimentation | 5 V via USB Type-C |
| Température de fonctionnement | *cf. datasheet Espressif ESP32-S3* |
| Tension des GPIO | **3,3 V** (high ≈ 3,3 V / low ≈ 0 V) |

### 2.1 Fonctions embarquées

- **W5500** : contrôleur Ethernet matériel avec pile TCP/IP interne (débite 10/100 Mbps sans charge CPU).
- **H1102NLT** : transformateur de couplage Ethernet (isolation galvanique).
- **JW5060** : régulateur de tension.
- **USB Type-C** : alimentation, débogage et téléchargement du firmware.
- **Emplacement TF/microSD** : stockage local (système de fichiers **FAT32 uniquement** ; exFAT et NTFS non supportés).
- **Connecteur d'extension PoE** : conforme **IEEE 802.3af**, alimenté par le câble réseau si un module PoE est connecté.
- **Interface caméra** : compatible **OV5640** et autres capteurs grand public.
- **Interface Pico-compatible** : permet de brancher certaines cartes d'extension Raspberry Pi Pico.
- **Boutons** : BOOT et RESET. **Voyants** : ACT et LINK.

### 2.2 Antenne

- Antenne **céramique** montée par défaut.
- Connecteur **IPEX** (Gen 1) réservé : pour l'utiliser, il faut **ressouder la résistance 0 Ω** de la position verticale (céramique) vers la position horizontale (IPEX).

---

## 3. Connectique

| Repère | Élément | Rôle |
|---|---|---|
| 1 | ESP32-S3R8 | Processeur dual-core 240 MHz |
| 2 | W25Q128 | Flash 16 Mo (programme + données) |
| 3 | W5500 | Contrôleur Ethernet |
| 4 | H1102NLT | Transformateur Ethernet |
| 5 | JW5060 | Régulateur de tension |
| 6 | Port USB Type-C | Téléchargement du programme + alimentation |
| 7 | Port RJ45 | Ethernet 10/100 Mbps auto-négociation |
| 8 | Connecteur d'extension PoE | Reçoit un module PoE IEEE 802.3af |
| 9 | Interface caméra | Compatible OV5640 |
| 10 | Connecteur antenne IPEX | Réservé — activation par soudure |
| 11 | Antenne céramique | Active par défaut |
| 12 | Emplacement carte TF | Stockage local |
| 13 | Bouton BOOT | Mode téléchargement |
| 14 | Voyant ACT | Indicateur d'activité |
| 15 | Voyant LINK | Indicateur de liaison réseau |
| 16 | Bouton RESET | Réinitialisation |

---

## 4. Occupation des GPIO — section critique

Avant toute conception, il faut connaître les broches **déjà occupées** par les périphériques embarqués, faute de quoi on provoque des conflits de bus.

### 4.1 Broches libres / utilisables

```
const int gpio_pin[] = { 21, 17, 16, 18, 15, 3, 2, 1, 0, 44,  43, 38, 39, 40, 41, 42, 45, 47, 48};
```

### 4.2 Répartition par fonction

| Fonction | Broches | Conséquence |
|---|---|---|
| **Non disponibles** | GPIO33 → GPIO37 | Occupées par des fonctions internes, **inexploitables en GPIO** |
| **Entrée seule** | GPIO46 | Ne peut pas être utilisée en sortie |
| **LED WS2812** | GPIO21 | RGB onboard — éviter pour un capteur |
| **W5500 (Ethernet)** | MISO 12, MOSI 11, SCLK 13, CS 14, RST 9, INT 10 | Ne pas réutiliser pour un capteur |
| **Carte TF (SPI)** | CS 4, MOSI 6, MISO 5, SCLK 7 | Libres si aucune carte TF |
| **Caméra** | EN 8, VSYNC 1, HREF 2, XCLK 3, PCLK 39, SIOD 48, SIOC 47, D7 18, D6 15, D5 38, D4 40, D3 42, D2 46, D1 45, D0 41 | Bloquées si une caméra est montée |

### 4.3 Broches recommandées pour un capteur de température

| Broche | Disponibilité | Usage recommandé |
|---|---|---|
| **GPIO16** | Libre | **1-Wire (DS18B20)** — avec résistance de tirage 4,7 kΩ vers 3,3 V |
| **GPIO17** | Libre | 1-Wire secondaire / 2ᵉ capteur (bus multi-capteurs en parallèle) |
| **GPIO43 / GPIO44** | Libres | Bus I²C (capteur MIMO, AHT20, BME280…) |
| GPIO0 | Libre mais câblée au bouton BOOT | À éviter (risque d'extinction de la carte) |
| GPIO4-7 | Libres si pas de carte TF | SPI (capteur type MAX6675) |

> **Attention — recouvrement de fonctions :** GPIO1, 2, 3, 15, 18, 38, 39, 40, 41, 42, 45, 46, 47, 48 sont listés comme libres dans le test d'E/S mais sont **aussi** affectés à l'interface caméra. Toute utilisation de ces broches impose de **retirer la caméra**.

---

## 5. Environnement de développement (Arduino)

### 5.1 Prérequis

- **Arduino IDE** avec le paquet de support **Arduino ESP32 version 2.0.12 ou ultérieure**.
- Carte sélectionnée dans le Boards Manager : **`ESP32S3 Dev Module`**.

### 5.2 Réglages du projet (menu « Outils »)

| Paramètre | Valeur |
|---|---|
| Board | `ESP32S3 Dev Module` |
| USB CDC On Boot | `Enabled` |
| CPU Frequency | `240MHz (WiFi)` |
| Flash Size | `16MB (128Mb)` |
| Partition Scheme | `Huge APP (3MB No OTA / 1MB SPIFFS)` |
| PSRAM | `OPI PSRAM` |
| Upload Mode | `UART0 / Hardware CDC` |
| Upload Speed | `921600` |
| USB Mode | `Hardware CDC and JTAG` |

### 5.3 Bibliothèques nécessaires

| Bibliothèque | Rôle | Installation |
|---|---|---|
| `Ethernet` | Exemples DHCP / IP statique du W5500 | Hors ligne (fournie) |
| `Ethernet_Generic` | Bascule vers IP statique si le DHCP échoue | Hors ligne (fournie) |
| `ETHClass2` | Exemple caméra sous Ethernet (Arduino ESP32 2.x) | Hors ligne (fournie) |
| `Adafruit_NeoPixel` | Pilotage de la LED WS2812 | En ligne ou hors ligne |
| `ESP32-BLE-Keyboard` | Fonctionnalités clavier Bluetooth | En ligne ou hors ligne |

> Un second environnement, **ESP-IDF**, est également supporté par le fabricant (voir documentation dédiée).

### 5.4 Exemples de référence fournis par Waveshare

| N° | Exemple | Description |
|---|---|---|
| 01 | `IO_Test` | Séquence haut/bas sur 19 GPIO, intervalle 300 ms |
| 02 | `RGB_LED` | Pilotage WS2812 via GPIO21 (aplats, dégradé, chenille) |
| 03 | `SD_Card` | Montage TF, création/lecture/écriture de fichier (FAT32) |
| 04 | `BLE_Keyboard` | émulation clavier Bluetooth |
| 05 | `WiFi_AP` | Point d'accès `ESP32-S3-ETH` / mot de passe `88888888` |
| 06 | `WiFi_STA` | Connexion Wi-Fi + serveur HTTP (commande GPIO18) |
| 07 | `WiFi_DHCP` | Obtention d'IP par DHCP en Wi-Fi |
| 08 | `WIFI_StaticIP` | IP statique en Wi-Fi |
| 09 | `WIFI_Web_CAM` | Serveur caméra temps réel sur Wi-Fi |
| **10** | **`ETH_DHCP`** | **Obtention d'IP par DHCP en Ethernet — base de notre projet** |
| **11** | **`ETH_StaticIP`** | **IP statique en Ethernet — indispensable pour la supervision MQTT** |
| **12** | **`ETH_MAC_DHCP_StaticIP`** | **MAC personnalisée + repli IP statique (`192.168.1.177`) — robuste en production** |
| 13 | `ETH_Web_CAM` | Serveur caméra sur Ethernet |

### 5.5 Récupération du firmware

- Exemple avec bibliothèques : `ESP32-S3-ETH-Demo.zip` (dossier `Arduino/examples`)
- Assistant de débogage série : **SSCOM 5.13.1**

---

## 6. Points d'attention et contraintes

| # | Contrainte | Conséquence / solution |
|---|---|---|
| 1 | **Aucun capteur de température embarqué** sur la carte | Le ESP32-S3-ETH **ne mesure pas la température** : il faut **ajouter un capteur externe** (DS18B20 en 1-Wire ou AHT20 en I²C) |
| 2 | Capteur interne du ESP32-S3 | Existe mais mesure la **température de la puce** (étalonnage interne), **inexploitable** pour l'ambiance d'une salle |
| 3 | GPIO33-37 inutilisables | Contournement impossible ; report sur d'autres broches |
| 4 | GPIO46 en entrée seule | Pas de commande de relais ou de LED d'alarme |
| 5 | Conflit caméra / GPIO | Ne pas mixér caméra et capteurs sur les mêmes broches |
| 6 | Carte TF en FAT32 uniquement | Reformater les cartes SDXC ≥ 64 Go (attention : effacement des données) |
| 7 | Accès TF par SDIO | Supporté **uniquement en mode 1 ligne**, à configurer dans `menuconfig` |
| 8 | Échec de téléchargement du firmware | Maintenir **BOOT**, appuyer sur **RESET**, relâcher RESET puis BOOT |
| 9 | IP dynamique = perte de supervision | **Privilégier l'IP statique** (exemple 12) pour un objet en production |
| 10 | Absence de PoE sur la carte de base | Alimentation par USB Type-C, ou acheter le **module PoE** séparément |

---

## 7. Application au projet « Supervision thermique »

### 7.1 Architecture retenue

```
 Salle COPERNIC                        Salle FAURIEL
 ┌────────────────────┐                ┌────────────────────┐
 │ ESP32-S3-ETH #1    │                │ ESP32-S3-ETH #2    │
 │ DS18B20 ×2         │                │ DS18B20 ×2         │
 │ (GPIO16, 1-Wire)   │                │ (GPIO16, 1-Wire)   │
 │ IP : 192.168.1.11  │                │ IP : 192.168.1.12  │
 │ Client MQTT        │                │ Client MQTT        │
 └─────────┬──────────┘                └─────────┬──────────┘
           │ Ethernet (W5500) · PUBLISH MQTT      │
           └────────────────┬────────────────────┘
                            ▼
                  ┌──────────────────────────────┐
                  │ Serveur de supervision       │
                  │  ┌─────────────────┐          │
                  │  │ Broker MQTT     │          │
                  │  │ Mosquitto       │          │
                  │  │ TCP 1883 / WS   │          │
                  │  └────────┬────────┘          │
                  │           ▼                   │
                  │  ┌─────────────────┐          │
                  │  │ Collecteur MQTT │          │
                  │  │ seuils + BDD    │          │
                  │  └───┬─────────┬───┘          │
                  └──────┼─────────┼─────────────┘
                         ▼         ▼
                ┌────────────┐  ┌──────────────┐
                │ Alerte     │  │ Page Web de  │
                │ email      │  │ supervision  │
                │ (SMTP)     │  │ + écran      │
                └────────────┘  └──────────────┘
```

> **Note d’architecture :** la supervision repose sur une **page Web de supervision** alimentée par **MQTT** (broker Mosquitto). Le firewall assure le cloisonnement du VLAN Supervision entre le réseau des salles serveurs et le réseau du bureau DSI. Le serveur de supervision est une **VM Debian installée dans la salle machine de Fauriel** (aile B), joignable depuis Copernic par la liaison Ethernet inter-sites (~2 km).
>
> **Traitement des seuils :** l’application des règles de seuil et l’envoi de l’email sont réalisés par le **collecteur MQTT du serveur**, et non par le navigateur. L’alerte est donc émise même si l’écran du bureau informatique est éteint, et aucun doublon d’email n’est possible.

**Pourquoi cette carte plutôt qu’un capteur réseau dédié ?**

- Ethernet **filaire** : indispensable en salle serveur (le Wi-Fi n’est pas retenu en zone critique).
- **Coût** bien inférieur à un capteur industriel avec agent de supervision (± 30 € contre plusieurs centaines).
- **Doubleusage** : Ethernet + Wi-Fi de secours, OLED via carte Pico d’extension, LED RGB d’alerte locale.
- **8 Mo de PSRAM** : largement suffisant pour un tampon d’historique et la pile réseau (Ethernet W5500 + client MQTT).

### 7.2 Chaîne de fonctionnement

1. Le **DS18B20** mesure la température et la transmet sur le bus 1-Wire au microcontrôleur.
2. L’ESP32 lit la valeur toutes les minutes, la met en forme en **JSON** et la **publie en MQTT** (QoS 1, retain) vers le broker Mosquitto, sur le topic `temperature/<salle>/<capteur>`.
3. Le **broker Mosquitto** reçoit le message, le conserve grâce au drapeau **retain** et le distribue à ses abonnés.
4. Le **collecteur MQTT** du serveur (abonné à `temperature/#` et `statut/#`) reçoit le message, l’**écrit dans la base de données** (historique) et le **compare aux règles de seuil**.
5. En cas de température **critique**, le collecteur déclenche l’**alerte email** au service informatique via la passerelle mail (SMTP) et republie une alerte sur le topic `alerte/temperature`.
6. La **page Web de supervision**, connectée au broker en **WebSocket** (port 9001), reçoit le message en temps réel et affiche les 4 températures, l’état `online`/`offline` de chaque carte et l’historique (lu en base). Elle **n’envoie aucun email** : l’alerte est déjà traitée par le collecteur.

Un **Last Will and Testament** publié par chaque ESP32 sur `statut/<équipement>` (`offline`) permet à la page Web de signaler immédiatement la perte de connexion d’un capteur.

**Bibliothèques Arduino à ajouter côté ESP32 :**

| Bibliothèque | Rôle |
|---|---|
| `OneWire` | Pilotage du bus 1-Wire (DS18B20) |
| `DallasTemperature` | Lecture et conversion de la température |
| `PubSubClient` | Client MQTT (publication) |
| `ArduinoJson` | Sérialisation des messages JSON |
| `Ethernet` | Connexion au réseau via W5500 |

### 7.3 Cahier des charges technique du capteur

| Critère | Exigence |
|---|---|
| Plage de mesure | −55 °C à +125 °C (DS18B20) |
| Résolution | 0,0625 °C (12 bits),modifiable par programme |
| Précision | ± 0,5 °C (0 °C à +85 °C) |
| Périodicité d'échantillonnage | 1 minute *(à valider)* |
| Communication | Ethernet filaire 10/100 Mbps, IP fixe |
| Protocole de supervision | **MQTT 3.1.1** (TCP 1883) ; WebSocket 9001 pour la page Web |
| Alimentation | 5 V USB Type-C, ou module PoE IEEE 802.3af |
| Encombrement | Format carte de développement (*cf. image de dimensions Waveshare*) |

### 7.4 Partition des rôles (2 cartes)

| Élément | Salle Copernic | Salle Fauriel |
|---|---|---|
| Adresse IP | 192.168.1.11 | 192.168.1.12 |
| Capteur | T1 (entrée salle) | T3 (entrée salle) |
| Second point de mesure | T2 (arrière salle) — à confirmer | T4 (arrière salle) — à confirmer |

> **À valider :** la spécification du cahier des charges exige **au moins deux points de mesure par salle**. Une carte ne gérant qu'un seul bus 1-Wire ne peut couvrir ce besoin qu'avec deux capteurs sur le même bus (adressage par identifiant 8 bits, câblage en parallèle sur GPIO16 + 4,7 kΩ). À défaut, il faut **2 cartes par salle (soit 4 au total)**.

### 7.5 Adressage réseau du projet

| Équipement | Adresse IP | VLAN |
|---|---|---|
| ESP32-S3-ETH #1 (Copernic) | 192.168.1.11 | Supervision |
| ESP32-S3-ETH #2 (Fauriel) | 192.168.1.12 | Supervision |
| Serveur de supervision (Mosquitto + page Web) | 192.168.1.10 | Supervision |
| Poste du bureau informatique | Via DHCP | Bureau |

> Les trois équipements de supervision sont sur le même VLAN, séparé du réseau du bureau informatique par le firewall. Le broker Mosquitto n'accepte les connexions que sur ce VLAN (port 1883 en TCP, 9001 en WebSocket).

---

## 8. Ressources et documentation

| Type | Lien |
|---|---|
| Schématique de la carte | `files.waveshare.com/wiki/ESP32-S3-ETH/ESP32-S3-ETH-Schematic.pdf` |
| Pinout (image) | `docs.waveshare.com/assets/files/ESP32-S3-ETH-details-15-...webp` |
| Datasheet ESP32-S3 | `documentation.espressif.com/esp32-s3_datasheet_en.pdf` |
| Technical Reference Manual | `documentation.espressif.com/esp32-s3_technical_reference_manual_en.pdf` |
| Datasheet W5500 | `files.waveshare.com/wiki/common/W5500_ds_v110e.pdf` |
| Exemples avec bibliothèques | `files.waveshare.com/wiki/ESP32-S3-ETH/ESP32-S3-ETH-Demo.zip` |
| FAQ officielle ESP | `docs.espressif.com/projects/esp-faq/en/latest/index.html` |
| Store Waveshare | `waveshare.com/esp32-s3-eth.htm` |

---

## 9. Documents liés

- `Cahier des charges capteur de température.md` — cadrage du besoin, périmètre et synoptique
- `schemas/5.1-chaine-fonctionnelle.drawio` — synoptique général du système
- `schemas/5.2-cablage-ds18b20.drawio` — câblage DS18B20 / ESP32-S3-ETH (GPIO16)
- `schemas/5.3-architecture-logicielle.drawio` — architecture logicielle du serveur de supervision
- `schemas/5.4-implantation-sites.drawio` — implantation des sites Copernic et Fauriel
- `schemas/5.5-serveur-web.drawio` — pile serveur Web (Apache2 / PHP-FPM / BDD)
- `schemas/5.6-cas-utilisation.drawio` — diagramme de cas d'utilisation
- `images/recto.jpg` / `images/verso.jpg` — croquis de travail manuscrits (implantation, architecture, cas d'utilisation)

> Tous les schémas sont modifiables avec [app.diagrams.net](https://app.diagrams.net) ou l’extension VS Code *Draw.io Integration*.
