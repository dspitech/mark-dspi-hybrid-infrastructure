# Mark-DSPI - Infrastructure Hybride & Sécurisée

**Cahier des charges** - Centralisation, Automatisation et Pilotage d'une Agence de Marketing
**Nom de code du projet :** Mark-DSPI
**Cible :** Environnement PME Hybride (Azure Cloud & Local)


---

## Sommaire

1. [Contexte, Vision et Objectifs Métier](#1-contexte-vision-et-objectifs-métier)
2. [Architecture Technique Générale (Hybride)](#2-architecture-technique-générale-hybride)
3. [Organisation de l'Annuaire](#3-organisation-de-lannuaire--départements-utilisateurs-et-rôles)
4. [Politique de Sécurité (GPO)](#4-politique-de-sécurité--les-gpo-et-fonctionnalités-clients-désactivées)
5. [Gestion de Parc & Contrôle à Distance](#5-gestion-de-parc-déploiement-logiciel-et-contrôle-à-distance)
6. [Journalisation Centralisée & Dashboard](#6-journalisation-centralisée--dashboard-de-monitoring)
7. [Plan d'Industrialisation via Terraform](#7-plan-dindustrialisation-et-déploiement-via-terraform)
8. [Matrice de Sécurité Fondamentale](#8-matrice-de-sécurité-fondamentale-hardening)

---

## 1. Contexte, Vision et Objectifs Métier

Mark-DSPI est une agence de marketing digital en pleine croissance. Elle gère des campagnes publicitaires d'envergure, des assets graphiques volumineux, des bases de données clients hautement confidentielles et des outils d'analyse de données avancés.

Jusqu'à présent, la gestion des postes de travail et des accès aux fichiers s'effectuait de manière décentralisée, ce qui pose des risques critiques en matière de **sécurité**, de **gouvernance des données (RGPD)** et d'**efficacité opérationnelle**.

### Objectifs stratégiques

| Objectif | Description |
|---|---|
| **Centralisation absolue** | Unifier la gestion des identités, des accès et des politiques de sécurité à partir d'un unique serveur Linux centralisé faisant office de Contrôleur de Domaine (compatible Active Directory). |
| **Automatisation complète (IaC)** | Déployer l'intégralité du réseau cloud et des machines virtuelles d'infrastructure via des scripts Terraform. |
| **Management moderne & pilotage centralisé** | Permettre le déploiement de logiciels à distance, la prise de contrôle à distance sécurisée et la centralisation de tous les logs dans un dashboard unifié. |
| **Sécurité globale de haut niveau** | Implémenter le principe du moindre privilège, segmenter le réseau, sécuriser la connectivité locale/cloud, auditer chaque action informatique et durcir drastiquement la configuration des postes de travail. |

---

## 2. Architecture Technique Générale (Hybride)

L'architecture interconnecte de manière transparente les ressources cloud (Microsoft Azure) et les ressources sur site (locaux physiques de l'agence).

### 2.1 Configuration réseau & tunnel sécurisé

| Élément | Plage / Détail |
|---|---|
| Réseau Virtuel Azure (VNet) | `10.10.0.0/16` |
| Sous-réseau Serveurs (`Subnet-SRV`) | `10.10.1.0/24` — héberge le serveur central |
| Sous-réseau Clients Cloud (`Subnet-CLI`) | `10.10.2.0/24` — héberge les 2 VM Windows |
| Réseau Local Entreprise (On-Premises) | `192.168.1.0/24` — réseau physique de l'agence |

**Liaison hybride sécurisée** : un tunnel VPN de site à site (Site-to-Site), ou un serveur VPN dédié (WireGuard / OpenVPN installé sur le serveur Azure), permet à la machine physique locale d'obtenir une adresse IP virtuelle dans le sous-réseau Azure et de dialoguer sans exposition publique avec le contrôleur de domaine.

**Résolution de noms (DNS)** : le serveur central Samba 4 est configuré comme l'unique résolveur DNS primaire de toutes les machines du parc. Tout trafic vers le domaine `mark-dspi.local` est intercepté par ce serveur.

### 2.2 Dimensionnement des systèmes (matrice des VMs)

| Hostname | Emplacement | OS | Spécifications | Rôles et services principaux |
|---|---|---|---|---|
| `SRV-CENTRAL` | Azure Cloud | Ubuntu Server 24.04 LTS | Standard_B2s (2 vCPU, 4 Go RAM, 64 Go SSD) | Samba 4 AD DC, DNS, serveur Syslog (Grafana/Loki), serveur Ansible/WAPT |
| `CLI-WIN-CL01` | Azure Cloud | Windows 11 Professionnel | Standard_B2ms (2 vCPU, 8 Go RAM, 128 Go SSD) | Poste de travail Cloud 1 / Console RSAT Administrateur |
| `CLI-WIN-CL02` | Azure Cloud | Windows 11 Professionnel | Standard_B2ms (2 vCPU, 8 Go RAM, 128 Go SSD) | Poste de travail Cloud 2 / Graphiste / Concepteur |
| `CLI-WIN-LOC01` | Local PME | Windows 11 Professionnel | Machine physique (ex. Core i5, 16 Go RAM) | Poste de travail Local Direction / Connecté via client VPN |

---

## 3. Organisation de l'Annuaire : Départements, Utilisateurs et Rôles

L'organisation au sein de l'annuaire Samba 4 suit une structure hiérarchique stricte en Unités Organisationnelles (OU).

### 3.1 Structure des Unités Organisationnelles (OU)

```
mark-dspi.local/
├── PME_Groupes_Securite/
├── PME_Administrateurs/
└── PME_Departements/
    ├── Direction/
    ├── Creation_Design/
    ├── Marketing_Strategique/
    └── Ressources_Humaines/
```

### 3.2 Matrice des utilisateurs et comptes de test

| Département | Utilisateur (login) | Nom | Fonction |
|---|---|---|---|
| Direction | `mmartin` | Marc Martin | Directeur Général |
| Création & Design | `sleroy` | Sophie Leroy | Directrice Artistique |
| Marketing Stratégique | `tdurand` | Thomas Durand | Data Analyst Marketing |
| Ressources Humaines | `vdubois` | Valérie Dubois | Responsable RH |

### 3.3 Matrice d'administration (rôles RBAC)

La délégation des tâches évite l'utilisation abusive du compte administrateur global :

| Rôle | Groupe AD | Droits | Utilisateur |
|---|---|---|---|
| Admin Système & Réseau (Global) | `Group_Admin_Sys` | Contrôle total sur `SRV-CENTRAL`, gestion globale de l'annuaire, modification des GPO | `admin-tech` |
| Admin Support & Helpdesk | `Group_Support_Helpdesk` | Réinitialisation des mots de passe, jointure au domaine, accès au contrôle à distance | `support-tech` |
| Admin RH (gestion des comptes) | `Group_Admin_RH` | Création et modification des fiches utilisateurs dans l'OU `Ressources_Humaines` et `Marketing_Strategique` uniquement | `vdubois` |

---

## 4. Politique de Sécurité : Les GPO et Fonctionnalités Clients Désactivées

Afin de durcir (*hardening*) les machines de l'agence contre l'exfiltration de données, le téléchargement de malwares ou les modifications systèmes accidentelles, un ensemble massif de restrictions et de désactivations est imposé par GPO aux machines Windows.

### 4.1 Sécurité des comptes (niveau domaine)

| GPO | Règle |
|---|---|
| `GPO_Securite_MotsDePasse` | Longueur minimale de 12 caractères avec complexité requise. Historique des 5 derniers mots de passe interdit. |
| `GPO_Verrouillage_Session` | Verrouillage automatique de la session après 10 minutes d'inactivité. Demande obligatoire du mot de passe lors du réveil. |

### 4.2 Matrice des fonctionnalités désactivées sur les machines clientes (critique)

| Fonctionnalité Windows | Statut | Objectif de sécurité |
|---|---|---|
| Invite de commande (`cmd.exe`) & PowerShell | ❌ Désactivée | Empêche l'exécution de scripts malveillants (obfusqués) ou l'exploration locale du système de fichiers par un attaquant |
| Panneau de configuration & Paramètres | ❌ Désactivé | Empêche les utilisateurs de modifier les configurations réseau, d'ajouter des périphériques ou de modifier la sécurité |
| Éditeur du Registre (`regedit.exe`) | ❌ Désactivé | Bloque la modification des clés de registre critiques pour contourner la sécurité du système |
| Gestionnaire des tâches (`taskmgr.exe`) | ❌ Désactivé | Interdit l'arrêt forcé des agents de sécurité, de monitoring ou de l'antivirus de l'entreprise |
| Stockage amovible (clés USB / disques externes) | 🚫 Bloqué | Désactivation des droits de lecture/écriture (sauf souris/clavier). Bloque l'exfiltration de données clients et l'injection de malwares (type Rubber Ducky) |
| Windows Store & installations locales | ❌ Désactivé | Bloque le téléchargement d'applications non approuvées ou de jeux par les employés |
| Exécution automatique (Autorun/Autoplay) | ❌ Désactivée | Évite l'infection automatique de la machine lors de l'insertion ou de la connexion d'un périphérique/image disque |
| Cortana & télémétrie Windows | ❌ Désactivée | Désactivation des fonctionnalités de recherche cloud et réduction au strict minimum de l'envoi de données vers Microsoft (RGPD) |
| Partage de fichiers de proximité (Wi-Fi/Bluetooth) | ❌ Désactivé | Bloque le partage direct d'assets marketing sensibles entre machines en dehors des dossiers partagés surveillés |

### 4.3 Mappage réseau et identité visuelle

| GPO | Description |
|---|---|
| `GPO_Mappage_Lecteurs` | Montage automatique du lecteur réseau général `M: \\SRV-CENTRAL\Commun` et des sous-dossiers restreints par département (`D:` pour la création, `X:` pour la direction) |
| `GPO_Bureau_Marketing` | Fond d'écran institutionnel standardisé obligatoire |

---

## 5. Gestion de Parc, Déploiement Logiciel et Contrôle à Distance

Puisque les utilisateurs ne possèdent aucun droit d'installation (fonctionnalité désactivée par GPO), la gestion du parc logiciel est entièrement pilotée depuis `SRV-CENTRAL` via **WAPT Server** ou **Ansible**.

### 5.1 Déploiement applicatif contrôlé

- **Logiciels communs** : suite bureautique, navigateurs (Google Chrome et Firefox durcis), lecteur PDF.
- **Logiciels métier (ciblés)** : outils de PAO (Adobe Creative Cloud) pour la Création ; outils d'analyse de données (Power BI Desktop, Python runtimes) pour le Marketing.

### 5.2 Contrôle à distance et assistance technique

- Intégration d'un agent léger (ex. **MeshCentral** ou agent WAPT).
- **Sécurité** : consentement explicite de l'utilisateur obligatoire par pop-up avant toute prise en main. Journalisation exhaustive de chaque action de support.

---

## 6. Journalisation Centralisée & Dashboard de Monitoring

Pour piloter l'infrastructure, la pile technologique **Grafana + Loki + Promtail** est installée sur `SRV-CENTRAL`.

### 6.1 Collecte des événements Windows et Linux

Les agents (Promtail/Winlogbeat) sur les clients transmettent en continu les journaux vers le serveur central. Une attention particulière est portée sur :

- L'activation et la remontée des logs d'audit du pare-feu Windows Defender.
- Les événements d'élévation de privilèges (UAC) ou de tentatives d'accès aux fonctionnalités désactivées (ex. tentative d'ouverture de `cmd.exe`).
- Les journaux d'accès réseau (Samba 4 / Kerberos) pour repérer les anomalies d'authentification.

### 6.2 Structure du dashboard centralisé Grafana

| Volet | Contenu |
|---|---|
| **Sécurité** | Alertes sur connexions infructueuses, blocages de comptes et détection d'insertions de clés USB non autorisées |
| **Parc** | Suivi des mises à jour de sécurité Windows (WSUS / WAPT Status) |
| **Santé** | Utilisation des ressources physiques des machines clientes et du serveur |

---

## 7. Plan d'Industrialisation et Déploiement via Terraform

Le provisionnement initial de l'infrastructure Cloud dans Microsoft Azure est entièrement automatisé.

### 7.1 Fichiers du projet Terraform

| Fichier | Rôle |
|---|---|
| `provider.tf` | Spécification du provider `azurerm` |
| `variables.tf` | Paramètres d'environnement (régions, tailles de VM, mots de passe initiaux sécurisés) |
| `main.tf` | Groupes de ressources, VNet, Subnets, Network Security Groups (NSG) filtrant strictement les ports AD, VPN et logs, et création des VMs |
| `outputs.tf` | IPs privées et publiques requises pour l'accès |

### 7.2 Post-configuration automatique (`cloud-init.txt`)

Le script de post-configuration automatique applique les mises à jour, installe Samba 4, configure les pare-feux locaux et prépare l'environnement de logs dès le démarrage de la VM Linux.

---

## 8. Matrice de Sécurité Fondamentale (Hardening)

| Domaine | Mesure |
|---|---|
| **Isolation SSH** | Accès interdit depuis Internet, authentification uniquement par clé Ed25519 depuis le réseau VPN |
| **Moindre privilège au stockage (ACLs)** | Droits d'accès stricts basés sur les groupes de l'annuaire Linux Samba 4. Un utilisateur du pôle Création ne peut en aucun cas lire les répertoires de la Direction ou des RH |
| **Chiffrement des flux** | Chiffrement systématique de toutes les données en transit via TLS 1.3, VPN, et SMB 3.x (chiffrement des partages réseau) |

---

## 9. Fonctionnalités Complémentaires (Renforcement Professionnel)

### 9.1 Résilience & continuité

| Mesure | Description |
|---|---|
| Sauvegarde | Azure Backup ou snapshots planifiés du disque de `SRV-CENTRAL`, avec test de restauration documenté |
| Plan de reprise après sinistre (PRA/PCA) | RTO/RPO définis, procédure de restauration de l'annuaire Samba 4 |

### 9.2 Identité & accès

| Mesure | Description |
|---|---|
| Accès conditionnel | Politique restreignant les connexions des comptes à privilèges par plage horaire et par IP source |
| Gestion centralisée des secrets | Azure Key Vault ou HashiCorp Vault pour éviter les mots de passe en clair dans les scripts Terraform/Ansible |

### 9.3 Automatisation & DevOps

| Mesure | Description |
|---|---|
| Versionnement Git | Dépôt Git pour Terraform/Ansible avec pipeline CI/CD (GitHub Actions) exécutant `terraform plan` sur chaque pull request |
| Tests d'infrastructure | Terratest ou scripts de validation post-déploiement |

### 9.4 Supervision élargie

| Mesure | Description |
|---|---|
| Alerting actif | Alertes Grafana envoyées par email/Teams, en complément de l'affichage dashboard |
| Audit de sécurité | Scan de vulnérabilités périodique (OpenVAS/Nessus Essentials) et audit CIS Benchmark sur les postes Windows |

### 9.5 Gouvernance des coûts

| Mesure | Description |
|---|---|
| Budgets Azure | Budgets et alertes de consommation configurés dans Azure Cost Management dès le premier déploiement |
| Extinction automatique | Auto-shutdown programmé des VM en dehors des heures de test |

### 9.6 Conformité & documentation

| Mesure | Description |
|---|---|
| Charte informatique | Charte utilisateur signée (usage du SI, sanctions en cas de contournement des GPO) |
| Registre RGPD | Registre de traitement RGPD pour les données clients hébergées |

---

## 10. Adaptation au Compte Azure for Students

Le crédit étudiant (environ 100$, non renouvelable, quotas de VM limités) impose des ajustements par rapport à une architecture de production :

- **VPN Gateway Azure** (~140$/mois) n'est pas finançable — le tunnel WireGuard/OpenVPN auto-hébergé sur `SRV-CENTRAL` est la solution retenue pour la maquette.
- Réduire `CLI-WIN-CL01` / `CLI-WIN-CL02` à des tailles plus économiques (ex. `Standard_B1s`/`B2s`) pour les phases de test, et ne les allumer qu'à la demande.
- Vérifier les quotas de région disponibles sur le compte étudiant avant l'écriture de `main.tf` (certaines tailles de VM ou régions sont restreintes).
- Éviter Azure Bastion (coûteux) — l'accès SSH via le tunnel VPN est suffisant pour une maquette de test.

---

## 11. Schéma d'Architecture

```
┌─────────────────────────────── VNet Azure — 10.10.0.0/16 ───────────────────────────────┐
│                                                                                            │
│   ┌────────── Subnet-SRV — 10.10.1.0/24 ──────────┐   ┌────────── Subnet-CLI — 10.10.2.0/24 ──────────┐
│   │                                                 │   │                                                │
│   │   SRV-CENTRAL                                   │   │   CLI-WIN-CL01 — Poste RSAT admin             │
│   │   Ubuntu 24.04, AD DC, DNS                       │   │   CLI-WIN-CL02 — Poste graphiste              │
│   │   Syslog, Ansible/WAPT                           │   │                                                │
│   └───────────────────────────────────────────────┘   └────────────────────────────────────────────────┘
│                                    │                                                                      │
└────────────────────────────────── │ ────────────────────────────────────────────────────────────────────┘
                                     │  Tunnel VPN (WireGuard/OpenVPN)
                                     ▼
┌───────────────────────── Réseau local PME — 192.168.1.0/24 ─────────────────────────────┐
│                                                                                            │
│                          CLI-WIN-LOC01 — Poste Direction (physique)                       │
│                                                                                            │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

*Document interne — Mark-DSPI*
