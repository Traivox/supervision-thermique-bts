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
2. **Remonter** les données en temps réel sur la page Web de supervision.
3. **Déclencher une alerte** si un seuil de température défini est dépassé.
4. **Envoyer un email** en cas de dépassement critique.
5. **Permettre un affichage permanent** des températures dans le bureau informatique.

### Périmètre

**Inclus dans le projet**

- Installation de capteurs de température dans les deux salles serveurs.
- Raccordement des capteurs au réseau et connexion à la page Web de supervision.
- Alerte par email en cas de température trop élevée.
- Affichage des températures sur un écran dans le bureau informatique.

**Non inclus dans le projet**

- La réparation ou le remplacement des climatisations.
- La modification des installations électriques ou des équipements existants.
- Les autres locaux du bâtiment.

## 5. Synoptique

Schéma éditable fourni en fichier Draw.io : **`Synoptique.drawio`**

Chaîne fonctionnelle :

```
Salle COPERNIC          Salle FAURIEL
 ┌──────────────┐       ┌──────────────┐
 │ Capteur T1   │       │ Capteur T3   │
 │ Capteur T2   │       │ Capteur T4   │
 └───────┬──────┘       └───────┬──────┘
         └──────────┬───────────┘
                    ▼
         ┌─────────────────────┐
         │  Switch / Réseau    │
         └──────────┬──────────┘
                    ▼
         ┌─────────────────────┐
         │  Page Web           │
         │  de supervision     │
         │  (règles de seuil)  │
         └────┬────────────┬────┘
              ▼            ▼
      ┌────────────┐  ┌──────────────┐
      │ Alerte     │  │ Écran bureau │
      │ email      │  │ informatique │
      └────────────┘  └──────────────┘
```

**Légende / chaîne fonctionnelle :**

| Étape | Élément | Rôle |
|---|---|---|
| 1 | Capteurs de température (2 par salle) | Mesure de la température |
| 2 | Switch / réseau | Transport des données |
| 3 | Page Web de supervision | Affichage, seuils, alarmes |
| 4 | Notification email | Alerte au service informatique |
| 5 | Écran bureau | Affichage permanent |

**Seuils de température :** *(à valider)*

| Niveau | Température | Action |
|---|---|---|
| Normal | T < 27 °C | Aucune |
| Avertissement | 27 °C ≤ T < 35 °C | Alerte visuelle dans la page |
| Critique | T ≥ 35 °C | Alerte visuelle + email au service informatique |
