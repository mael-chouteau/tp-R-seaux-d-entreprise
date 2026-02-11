## TP — Conception et urbanisation de services réseau

### Contexte et enjeux

Dans ce TP, chaque étudiant conçoit et déploie une maquette réseau basée sur des équipements Cisco (routeurs 2800, switchs 2960, ASA 5512-X) associée à une stack de services Linux (Kea, Bind9, LLDAP, ProFTPD).

Les objectifs principaux sont :

- **Segmentation** : isolation des flux via VLAN (Data / Management) et routage IP.
- **Haute disponibilité logique** : routage dynamique (OSPF) entre routeurs et accès Internet redondé via un routeur de bordure (Core) et un ASA.
- **Services d’infrastructure** : adressage automatique (DHCP Kea), résolution de noms (Bind9), annuaire (LLDAP) et authentification centralisée pour ProFTPD.

La maquette est **paramétrable** à l’aide de deux variables :

- **\(X\)** : identifiant groupe (ex. 1, 2, 3…).
- **\(Y\)** : identifiant d’étudiant au sein de la rangée (ex. 1, 2, 3, 4…).

Toutes les adresses IP et noms devront être adaptés en conséquence.

---

### Objectifs pédagogiques

À l’issue du TP, l’étudiant doit être capable de :

- Configurer un **switch Cisco 2960** (VLAN, access, trunk).
- Configurer un **routeur Cisco 2800** (sous-interfaces 802.1Q, routage OSPF).
- Déployer ** DHCP**, **Bind9**, **LLDAP** et **ProFTPD** avec authentification LDAP.
- Créer un rapport structurée et produire des **preuves** (commandes, captures).

---

## I. Topologie — Description textuelle

Chaque rangée (groupe, \(X\)) dispose du matériel suivant :

- **5x Cisco 2800** (routeurs).
- **6x Cisco 2960** (switchs).
- **1x Cisco ASA 5512-X** (pare-feu de bordure).

Pour chaque étudiant identifié par \(Y\) :

- 1 **PC étudiant** connecté à 1 **switch 2960 dédié** (`SW_XY`).
- 1 **routeur 2800 dédié** (`R_XY`) relié au switch par un lien **trunk 802.1Q**.
- Les VLANs principaux sont :
  - **VLAN 10** : Data (poste de travail).
  - **VLAN 20** : Management (accès aux équipements, services, supervision).

Pour éviter les conflits en travaux pratiques, chaque étudiant doit déployer son propre lot de VMs (Ubuntu Server 24.04) :

- 1 **VM DNS/DHCP** : `dnsdhcp_XY` (Kea + Bind9).
- 1 **VM FTP/LDAP** : `ftpldap_XY` (LLDAP + ProFTPD).
- 1 **VM supervision** : `zabbix_XY` (Zabbix appliance 7.4.6).
- 1 **VM client** : `client_XY` (tests utilisateurs).

Chaque groupe dispose en plus d’une interconnexion commune :

- 1 réseau de transit inter-routeurs.
- 1 routeur Core.
- 1 ASA 5512-X vers Internet.

**Chemin typique des paquets (intra-groupe)** :

1. PC étudiant `PC_XY` relié au switch `SW_XY` via **un seul câble** (port access dans VLAN 10 Data ; VLAN 20 Mgmt accessible par IP distincte).
2. `SW_XY` est relié au routeur `R_XY` via un lien en **trunk 802.1Q**, transportant au minimum les VLANs 10 et 20.
3. `R_XY` assure le **routage inter-VLAN** (sous-interfaces dot1Q) et la connectivité vers le **réseau de transit** commun entre routeurs.
4. Tous les routeurs `R_XY` sont interconnectés via un **switch de transit** à un **routeur de bordure (Core)**.
_5. Le **routeur de bordure** est relié à l’**ASA 5512-X**, qui assure le **NAT** et le contrôle d’accès vers la zone Outside (Internet)._

**Schéma à venir :**

---

## II. Plan d’adressage (avec variables X et Y)

### Conventions d’adressage

- **LAN Data étudiant (VLAN 10)** : `10.X.Y.0/24`
  - Passerelle par défaut (sous-interface routeur) : `10.X.Y.254`
- **LAN Management étudiant (VLAN 20)** : `10.X.Y.0/24`
  - Passerelle de management (sous-interface routeur) : IP dédiée (ex. `10.X.Y.252` ou autre adresse réservée dans le /24).
- **Transit inter-routeurs (X)** : `172.16.X.0/24`
  - Adresses précises à définir par groupe pour chaque lien (ex. `.1` pour Core, `.Y` pour chaque `R_XY`, `.254` pour ASA inside).

### Tableau d’adressage — Modèle générique

#### Rappels de variables

- \(X\) : rangée / groupe (1, 2, 3, …)
- \(Y\) : étudiant (1, 2, 3, …)

#### Tableau principal

| Rôle / Équipement              | VLAN | Réseau / Masque     | Adresse IP (exemple)         | Passerelle      | Interface / Port               | Commentaire                                      |
|--------------------------------|------|----------------------|------------------------------|-----------------|--------------------------------|-------------------------------------------------|
| PC Data étudiant `PC_XY`       | 10   | 10.X.Y.0/24          | 10.X.Y.100                    | 10.X.Y.254      | `SW_XY` Fa0/1 (access)         | Adresse fournie par DHCP Kea (scope Data)       |
| SW_XY (SVI Mgmt)               | 10   | 10.X.Y.0/24          | 10.X.Y.253                   | 10.X.Y.254      | Vlan 10                        | IP de management du switch                      |
| R_XY – sous-if Data            | 10   | 10.X.Y.0/24          | 10.X.Y.254                   | —               | `G0/0.10` (dot1Q 10)           | Passerelle Data étudiante                       |
| R_XY – sous-if Mgmt            | 20   | 10.X.Y.0/24          | 10.X.Y.252                   | —               | `G0/0.20` (dot1Q 20)           | Passerelle Management (adresse distincte du .254 Data) |
| R_XY – interface Transit       | —    | 172.16.X.0/24        | 172.16.X.Y              | —               | `G0/1`                         | Lien vers switch de transit                     |
| Routeur Core – vers transit    | —    | 172.16.X.0/24        | 172.16.X.254                 | —               | `G0/0`                         | Vers switch de transit (gateway des routeurs R_XY) |
| ASA Inside                     | —    | 192.168.X.0/30       | 192.168.X.2                  | —               | `inside`                       | Lien p2p Core–ASA (Core = 192.168.X.1)          |
| ASA Outside                    | —    | (réseau ISP labo)    | donné par enseignant         | passerelle ISP  | `outside`                      | Interface outside, vers Internet                |
| VM DNS/DHCP                    | 10   | 10.X.Y.0/24          | 10.X.Y.1                     | 10.X.Y.254      | NIC vers VLAN 10 (ou trunk)    | Kea + Bind9                                     |
| VM supervision (Zabbix)        | 10   | 10.X.Y.0/24          | 10.X.Y.2                     | 10.X.Y.254      | NIC vers VLAN 10 (ou trunk)    | Zabbix serveur                                  |
| VM FTP/LDAP                    | 10   | 10.X.Y.0/24          | 10.X.Y.3                     | 10.X.Y.254      | NIC vers VLAN 10 (ou trunk)    | LLDAP + ProFTPD (auth LDAP)                     |
| VM client Linux `client_XY`    | 10   | 10.X.Y.0/24          | 10.X.Y.4 (DHCP ou statique) | 10.X.Y.254      | NIC vers VLAN 10 (ou trunk)    | Poste de test supplémentaire (ping/dig/FTP…)    |

> **Exemple chiffré** : pour \(X = 1\), \(Y = 3\)
> - LAN Data : `10.1.3.0/24`, passerelle `10.1.3.254`
> - LAN Mgmt : `10.1.20.0/24`, passerelle `10.1.20.254`
> - Transit : `172.16.1.0/24`, `R_13` = `172.16.1.13`, Core = `172.16.1.1`, ASA inside = `172.16.1.254`

---

## III. Guide de configuration pas-à-pas (Cisco IOS / ASA)

### III.0 Remise à zéro et sauvegarde des configurations

Avant de commencer la configuration, assurez-vous que les équipements sont dans un état propre et que vous savez **sauvegarder** puis **restaurer** vos configurations.

#### III.0.1 Remettre un switch/routeur Cisco à zéro

Depuis le mode privilégié :

```plaintext
enable
erase startup-config
reload
```

Au redémarrage, répondre **No** si l’assistant de configuration interactive (`auto-config` ou `setup`) se lance.

#### III.0.2 Sauvegarder la configuration en NVRAM

Une fois la configuration terminée (ou à un point stable), sauvegardez-la dans la NVRAM :

```plaintext
enable
write memory
! ou
copy running-config startup-config
```

#### III.0.3 Exporter / restaurer une configuration via TFTP (optionnel)

Si un serveur TFTP est disponible sur votre VLAN Data, vous pouvez archiver vos configs :

```plaintext
! Sauvegarder la configuration courante vers un serveur TFTP
enable
copy running-config tftp:
 Address or name of remote host []? 10.X.Y.1
 Destination filename [running-config]? R_XY.cfg

! Restaurer une configuration depuis un serveur TFTP
enable
copy tftp: running-config
 Address or name of remote host []? 10.X.Y.1
 Source filename []? R_XY.cfg
```

> **Bonnes pratiques** :
> - Sauvegarder régulièrement (`write memory`) à chaque étape importante.
> - Conserver une **copie propre minimale** (config de base) et une **copie finale** (config complète commentée).

### III.1 Switch Cisco 2960 — VLANs, Access, Trunk

#### III.1.1 Préparation du switch

```plaintext
enable
erase startup-config
reload

enable
configure terminal
hostname SW_XY

! Mot de passe console / vty (à adapter)
line console 0
 password cisco
 login
exit

line vty 0 4
 password cisco
 login
exit

! Sauvegarde
end
write memory
```

#### III.1.2 Création des VLANs

```plaintext
configure terminal

vlan 10
 name DATA_XY
vlan 20
 name MGMT_X

end
write memory
```

#### III.1.3 Interface de management du switch (SVI Vlan10)

```plaintext
configure terminal

interface Vlan10
 ip address 10.X.Y.253 255.255.255.0
 no shutdown

ip default-gateway 10.X.Y.254

end
write memory
```

> Cette interface permet d’administrer `SW_XY` en IPv4 (SSH, HTTP/HTTPS si activé) via le réseau Data `10.X.Y.0/24`.

#### III.1.4 Configuration du port access pour le PC

Supposons que le PC de l’étudiant soit connecté sur `FastEthernet0/1` :

```plaintext
configure terminal

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown

end
write memory
```

> **Troubleshooting (classiques)**
> - Vérifier que le port n’est pas `shutdown` : `show interfaces status`, `show running-config interface Fa0/1`.
> - Vérifier que le VLAN 10 existe : `show vlan brief`.
> - Si le PC ne reçoit pas d’adresse IP, vérifier que la carte réseau est bien configurée en DHCP et que Kea voit les requêtes.

#### III.1.5 Configuration du port trunk vers le routeur

Supposons que le lien vers `R_XY` soit `FastEthernet0/24` :

```plaintext
configure terminal

interface FastEthernet0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20
 no shutdown

end
write memory
```

> **Troubleshooting**
> - Mismatch access/trunk : si côté routeur vous utilisez des sous-interfaces dot1Q, le port côté switch DOIT être en **trunk**.
> - Vérifier avec `show interfaces trunk` que les VLANs 10 et 20 sont bien autorisés.

---

### III.2 Routeur Cisco 2800 — Sous-interfaces & OSPF

#### III.2.1 Sous-interfaces pour VLAN 10 et VLAN 20

On suppose :

- Lien trunk vers switch : `GigabitEthernet0/0`.
- VLAN 10 (Data) / VLAN 20 (Mgmt).

```plaintext
enable
configure terminal

hostname R_XY

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.X.Y.254 255.255.255.0
 no shutdown

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.X.20.254 255.255.255.0
 no shutdown

end
write memory
```

#### III.2.2 Interface de transit vers le Core

On suppose :

- Lien vers switch de transit sur `GigabitEthernet0/1`.
- Adresse du routeur `R_XY` sur transit : `172.16.X.(10+Y)`, masque `/24`.

```plaintext
configure terminal

interface GigabitEthernet0/1
 ip address 172.16.X.(10+Y) 255.255.255.0
 no shutdown

end
write memory
```

#### III.2.3 Configuration OSPF

Objectif : annoncer les réseaux Data, Mgmt et Transit dans une **même area OSPF** (par exemple area 0).

```plaintext
configure terminal

router ospf 1
 router-id 1.X.Y.Y      ! à adapter (exemple)
 network 10.X.Y.0 0.0.0.255 area 0
 network 10.X.20.0 0.0.0.255 area 0
 network 172.16.X.0 0.0.0.255 area 0

end
write memory
```

> **Troubleshooting OSPF**
> - Vérifier les voisins : `show ip ospf neighbor`.
> - Vérifier la table de routage : `show ip route ospf`.
> - En cas d’absence de routes, vérifier **wildcards** et masques : le `network 172.16.X.0 0.0.0.255 area 0` doit correspondre exactement au réseau utilisé sur l’interface.
> - Ne pas oublier `no shutdown` sur les interfaces physiques et sous-interfaces.

---

### III.3 ASA 5512-X — Interfaces, NAT, Routes

#### III.3.1 Paramétrage de base des interfaces

Hypothèses :

- Interface inside : lien point-à-point vers le routeur Core, réseau `192.168.X.0/30` (Core = `192.168.X.1`, ASA inside = `192.168.X.2`).
- Interface outside : réseau fourni par l’ISP de labo (ex. `192.0.2.0/24`).

```plaintext
enable
configure terminal

hostname ASA_CORE_X

interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 192.0.2.2 255.255.255.0
 no shutdown

interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.X.2 255.255.255.252
 no shutdown

end
write memory
```

#### III.3.2 Route par défaut et routage

```plaintext
configure terminal

route outside 0.0.0.0 0.0.0.0 192.0.2.1

end
write memory
```

> Côté routeur Core, configurez une route par défaut pointant vers l’ASA inside (`ip route 0.0.0.0 0.0.0.0 172.16.X.254`) ou annoncez une défaut via OSPF selon le design retenu.

#### III.3.3 NAT dynamique (Inside → Outside)

Exemple de NAT dynamique pour permettre aux réseaux `10.X.0.0/16` (ou vos sous-réseaux précis) d’accéder à Internet :

```plaintext
configure terminal

object network OBJ_INSIDE_NET
 subnet 10.X.0.0 255.255.0.0
 nat (inside,outside) dynamic interface

end
write memory
```

> **Troubleshooting ASA**
> - Vérifier la table de routage : `show route`.
> - Vérifier le NAT : `show nat`, `show xlate`.
> - En cas de blocage, vérifier l’existence d’ACL sur l’interface outside si vous avez activé un filtrage explicite.

---

## IV. Déploiement des services (Kea, Bind9, LLDAP, ProFTPD, Zabbix)

Les services sont répartis sur plusieurs VMs Linux (ex. Ubuntu) situées dans le LAN Data :

- VM DNS/DHCP.
- VM supervision Zabbix.
- VM FTP/LDAP.
- VM client.

### IV.1 Plan général (par étudiant)

- Système recommandé : **Ubuntu Server 24.04 LTS** pour `dnsdhcp_XY`, `ftpldap_XY`, `client_XY`.
- **VM DNS/DHCP** : `dnsdhcp_XY`, IP `10.X.Y.1` (LAN Data).
- **VM supervision (Zabbix)** : `zabbix_XY`, IP `10.X.Y.2` (appliance Zabbix 7.4.6).
- **VM FTP/LDAP** : `ftpldap_XY`, IP `10.X.Y.3`.
- **VM client** : `client_XY`, IP en DHCP dans le scope Data (par exemple `10.X.Y.4`).
- Passerelle par défaut des VMs : `10.X.Y.254`.
- Noms de domaine : `x.lab.local` (à adapter à votre groupe).
- Carte réseau VM : mode **bridged** vers le VLAN Data du poste étudiant (ou port-group équivalent en hyperviseur).
---

### IV.2 Kea DHCP (JSON)

Kea utilise un fichier de configuration **JSON** pour le serveur DHCPv4. La configuration officielle par défaut (ex. `/etc/kea/kea-dhcp4.conf`) est structurée autour du bloc `"Dhcp4"` et peut servir de base : interfaces d’écoute, base des baux, sous-réseaux, options et éventuellement réservations.

#### Installation (Ubuntu / Debian)

```bash
sudo apt-get update
sudo apt-get install -y kea-dhcp4-server
```

Le paquet installe un fichier d’exemple. **Sauvegardez-le** puis remplacez ou adaptez le contenu comme ci-dessous.

```bash
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak
```

#### Structure du fichier de configuration

Le serveur DHCPv4 Kea ne lit que la section **`"Dhcp4"`**. Les éléments essentiels sont :

| Élément | Rôle |
|--------|------|
| `interfaces-config` | Interfaces sur lesquelles Kea écoute (ex. `eth0`). À renseigner obligatoirement. |
| `control-socket` | Socket Unix pour les commandes de gestion (config-reload, statistiques, etc.). |
| `lease-database` | Stockage des baux (type `memfile` = fichier CSV en mémoire). |
| `expired-leases-processing` | Réclamation et purge des baux expirés. |
| `renew-timer`, `rebind-timer`, `valid-lifetime` | Durées de vie des baux (optionnel au niveau global). |
| `option-data` | Options DHCP globales (peuvent être surchargées par sous-réseau ou réservation). |
| `subnet4` | Liste des sous-réseaux avec `id`, `subnet`, `pools` et `option-data` par sous-réseau. |
| `reservations` | (Par sous-réseau) Réservations d’adresses par MAC, client-id, DUID, etc. |
| `loggers` | Sortie et niveau des logs (stdout, fichier, syslog). |

#### Exemple adapté au TP (variables X et Y)

Remplacez `X` et `Y` par les valeurs de votre groupe/étudiant. L’option de passerelle doit s’appeler **`routers`** (option standard DHCP, code 3).

**Fichier** : `/etc/kea/kea-dhcp4.conf`

```json
{
  "Dhcp4": {
    "interfaces-config": {
      "interfaces": [ "eth0" ]
    },
    "control-socket": {
      "socket-type": "unix",
      "socket-name": "/run/kea/kea4-ctrl-socket"
    },
    "lease-database": {
      "type": "memfile",
      "lfc-interval": 3600
    },
    "expired-leases-processing": {
      "reclaim-timer-wait-time": 10,
      "flush-reclaimed-timer-wait-time": 25,
      "hold-reclaimed-time": 3600,
      "max-reclaim-leases": 100,
      "max-reclaim-time": 250,
      "unwarned-reclaim-cycles": 5
    },
    "renew-timer": 900,
    "rebind-timer": 1800,
    "valid-lifetime": 3600,
    "option-data": [
      { "name": "domain-name-servers", "data": "10.X.Y.1" },
      { "code": 15, "data": "x.lab.local" }
    ],
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.X.Y.0/24",
        "pools": [ { "pool": "10.X.Y.10 - 10.X.Y.99" } ],
        "option-data": [
          { "name": "routers", "data": "10.X.Y.254" },
          { "name": "domain-name-servers", "data": "10.X.Y.1" },
          { "name": "domain-name", "data": "x.lab.local" }
        ],
      }
    ],
    "loggers": [
      {
        "name": "kea-dhcp4",
        "output_options": [ { "output": "stdout", "pattern": "%-5p %m\n" } ],
        "severity": "INFO",
        "debuglevel": 0
      }
    ]
  }
}
```

- **Interfaces** : indiquez le nom réel de l’interface (ex. `eth0`, `ens18`). Avec plusieurs sous-réseaux relayés, Kea peut n’écouter que sur une seule interface ; les paquets relayés sont alors identifiés par l’adresse du relay.
- **Options** : `routers` = passerelle par défaut ; `domain-name-servers` = IP du serveur DNS (VM DNS/DHCP `10.X.Y.1`) ; `domain-name` = domaine de recherche.

#### Validation et redémarrage

```bash
# Vérifier la syntaxe du fichier (Kea peut le faire au démarrage)
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf

sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

Recharger la configuration sans redémarrer le service (socket de contrôle) :

```bash
sudo systemctl reload kea-dhcp4-server
```

#### Couplage avec le réseau

- Si la VM DNS/DHCP est **dans le même VLAN** que les clients (VLAN 10 Data), Kea reçoit directement les requêtes DHCP sur `eth0` ; aucun relay n’est nécessaire pour ce VLAN.
- Si des clients sont dans un autre segment (ex. VLAN 20 Mgmt) ou derrière un routeur, configurez un **IP helper** sur le routeur Cisco vers l’IP de la VM Kea :

```plaintext
interface GigabitEthernet0/0.10
 ip helper-address 10.X.Y.1
interface GigabitEthernet0/0.20
 ip helper-address 10.X.Y.1
```

> **Troubleshooting** : en cas d’échec au démarrage, vérifier les logs (`journalctl -u kea-dhcp4-server -f`) et que les plages `pool` sont bien incluses dans les `subnet` déclarés. Vérifier aussi que l’option s’appelle bien `routers` et non `router`.

---

### IV.3 Bind9 (DNS)

Installation :

```bash
sudo apt-get install -y bind9
```

Configuration minimale :

- Fichier `named.conf.local` pour déclarer vos zones.
- Zone directe `x.lab.local`.
- Zone inverse pour `10.X.y.0/16` ou un sous-ensemble pertinent.

Exemple de `named.conf.local` :

```plaintext
zone "x.lab.local" {
  type master;
  file "/etc/bind/db.x.lab.local";
};

zone "X.10.in-addr.arpa" {
  type master;
  file "/etc/bind/db.10.X";
};
```

Exemple très simplifié de zone directe (`/etc/bind/db.x.lab.local`) :

```plaintext
$TTL 86400
@   IN SOA  ns1.x.lab.local. admin.x.lab.local. (
        2026010101 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum

@       IN NS   ns1.x.lab.local.
ns1     IN A    10.X.Y.1
srv     IN A    10.X.Y.1
core    IN A    172.16.X.1
asa     IN A    172.16.X.254
```

Vérifications :

```bash
sudo named-checkconf
sudo named-checkzone x.lab.local /etc/bind/db.x.lab.local
sudo systemctl restart bind9
dig @10.X.Y.1 srv.x.lab.local
```

---

### IV.4 LLDAP (depuis [le guide](https://blog.stephane-robert.info/docs/services/identite/lldap/))

## Installation des pré-requis
### 1. SQLite3

LLDAP utilise **SQLite** comme moteur de base de données par défaut. Pour l'installer (sur Debian/Ubuntu) :

```bash
sudo apt update && sudo apt install sqlite3 libsqlite3-dev
```

### 2. mkcert

Indispensable pour générer localement des certificats **TLS** auto-signés de manière simple :

```bash
sudo apt install mkcert
```

### 3. Outils LDAP

Pour tester et requêter votre futur serveur, installez les utilitaires clients :

```bash
sudo apt install ldap-utils
```

---

## Installation de LLDAP
### Ajout du dépôt (Repository)
Exécutez ces commandes pour configurer la clé GPG et la source du dépôt :

```bash
# Ajout de la clé de signature
curl -fsSL https://download.opensuse.org/repositories/home:Masgalor:LLDAP/xUbuntu_24.04/Release.key | \
  gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/lldap.gpg > /dev/null

# Ajout du dépôt LLDAP
echo "deb http://download.opensuse.org/repositories/home:/Masgalor:/LLDAP/xUbuntu_24.04/ /" | \
  sudo tee /etc/apt/sources.list.d/home:Masgalor:LLDAP.list > /dev/null

```

### Installation du paquet

Une fois le dépôt configuré, mettez à jour votre cache et installez LLDAP :

```bash
sudo apt update
sudo apt install lldap

```

---

## Vérification de l'installation

Pour confirmer que tout est opérationnel, vérifiez la version installée :

```bash
lldap --version

```

> **Note :** Si un numéro de version s'affiche sans erreur, votre serveur LLDAP est prêt à être configuré !

---

### IV.5 ProFTPD (authentification LDAP)

Objectif : authentifier les utilisateurs FTP sur LLDAP et isoler chaque étudiant dans son répertoire.

#### Installation des paquets

```bash
sudo apt-get install -y proftpd-basic proftpd-mod-ldap
```

Activez explicitement le module LDAP :

```bash
sudo sed -i 's/^#\?LoadModule mod_ldap.c/LoadModule mod_ldap.c/' /etc/proftpd/modules.conf
```

Créez un fragment de configuration dédié :

```bash
sudo nano /etc/proftpd/conf.d/lldap.conf
```

Exemple de configuration (à adapter à votre arbre LDAP) :

```plaintext
<IfModule mod_ldap.c>
  LDAPServer ldap://10.X.Y.3:3890
  LDAPAuthBinds on
  LDAPBindDN "uid=admin,ou=people,dc=x,dc=lab,dc=local" "MotDePasseAdminLDAP"
  LDAPDefaultUID 2000
  LDAPDefaultGID 2000
  LDAPForceDefaultUID on
  LDAPForceDefaultGID on
  LDAPUsers "ou=people,dc=x,dc=lab,dc=local" "(uid=%u)"
  LDAPGroups "ou=groups,dc=x,dc=lab,dc=local" "(cn=%v)"
</IfModule>

ServerName "FTP_LDAP_XY"
DefaultRoot ~
RequireValidShell off
UseIPv6 off
PassivePorts 50000 50100

<Limit LOGIN>
  AllowAll
</Limit>
```

Points importants :

- `LDAPAuthBinds on` force une authentification réelle sur LDAP (pas juste une recherche).
- `DefaultRoot ~` confine l’utilisateur dans son home.
- `RequireValidShell off` évite de rejeter des comptes LDAP sans shell Unix classique.
- Pour un vrai chiffrement, préférez **FTPS explicite** (`TLSRequired on`) et un certificat local.

#### Préparer les répertoires des utilisateurs FTP

Si les homes ne sont pas créés automatiquement par LDAP, créez-les localement :

```bash
sudo mkdir -p /srv/ftp-homes
sudo chown root:root /srv/ftp-homes
sudo chmod 755 /srv/ftp-homes
```

Puis définissez un modèle de home dans LDAP (ex. `/srv/ftp-homes/%u`) ou créez les dossiers par utilisateur.

#### Redémarrage et vérification

```bash
sudo systemctl restart proftpd
sudo systemctl status proftpd
sudo journalctl -u proftpd -f
```

#### Tests fonctionnels minimaux

1. Vérifier la recherche LDAP depuis le serveur FTP :
   ```bash
   ldapsearch -x -H ldap://10.X.Y.3 -b "dc=x,dc=lab,dc=local" -W -D "uid=admin,ou=people,dc=x,dc=lab,dc=local" "(uid=admin)"
   ```
2. Tester une connexion FTP/FTPS depuis `client_XY`.
3. Effectuer `put` puis `get` d’un fichier.
4. Vérifier les logs ProFTPD et les permissions du home.

---

### IV.6 Zabbix 7.4.6 (appliance .ovf / .vmx)

Objectif : déployer rapidement un serveur de supervision pour chaque étudiant à partir de l’appliance officielle Zabbix.

> L’appliance est adaptée au TP et à l’évaluation, pas à une production critique.

#### IV.6.1 Choix de l’image

- `OVF` : import standard multi-hyperviseurs (vSphere, VirtualBox, certains outils KVM).
- `VMX` : import direct VMware Workstation / ESXi (fichiers VMware prêts à l’emploi).
- Version recommandée pour ce TP : **Zabbix 7.4.6**.

Téléchargement : choisir `zabbix_appliance-7.4.6-ovf.tar.gz` ou `zabbix_appliance-7.4.6-vmx.tar.gz` selon l’hyperviseur.

#### IV.6.2 Déploiement OVF (méthode générique)

1. Télécharger et extraire l’archive :
   ```bash
   tar -xzf zabbix_appliance-7.4.6-ovf.tar.gz
   ```
2. Importer le fichier `.ovf` dans l’hyperviseur.
3. Affecter la carte réseau au VLAN Data étudiant (réseau `10.X.Y.0/24`).
4. Démarrer la VM puis configurer l’IP statique `10.X.Y.2/24`, passerelle `10.X.Y.254`, DNS `10.X.Y.1`.

#### IV.6.3 Déploiement VMX (VMware)

1. Extraire l’archive :
   ```bash
   tar -xzf zabbix_appliance-7.4.6-vmx.tar.gz
   ```
2. Ouvrir le fichier `.vmx` dans VMware Workstation/ESXi.
3. Vérifier CPU/RAM/Disque (minimum conseillé en TP : 2 vCPU, 4 Go RAM).
4. Connecter l’interface réseau au bon segment VLAN Data.
5. Démarrer puis fixer l’IP `10.X.Y.2`.

#### IV.6.4 Initialisation Zabbix

Une fois la VM démarrée :

1. **Connexion console (shell de la VM)**  
   Depuis la console de l’hyperviseur ou en SSH sur la VM Zabbix :
   - Login système : `root`  
   - Mot de passe : `zabbix`  
   (ce sont les identifiants par défaut de l’appliance Zabbix ; ils doivent être changés ensuite).

2. **Vérifier les services Zabbix et Web**  
   Après connexion, si un **menu texte** de l’appliance apparaît, tapez d’abord :
   ```bash
   shell
   ```
   pour obtenir un vrai prompt Linux (`root@zabbix:~#`).  
   Ensuite, dans ce shell :
   ```bash
   systemctl status zabbix-server zabbix-agent nginx php-fpm mysql
   ```
   Les services doivent être en état `active (running)` ou équivalent.

3. **Accéder à l’interface Web**  
   Depuis un navigateur sur `client_XY` ou `PC_XY` :
   - URL : `http://10.X.Y.2/zabbix`
   - Identifiants par défaut :
     - Utilisateur : `Admin`  (attention à la majuscule)
     - Mot de passe : `zabbix`

4. **Finaliser l’assistant et sécuriser l’accès**  
   - Suivre l’assistant web si proposé (les paramètres de base MySQL sont déjà préconfigurés dans l’appliance).
   - Aller dans la section Administration / Users et **changer immédiatement** le mot de passe du compte `Admin`.

#### IV.6.5 Inventaire et supervision par étudiant

Chaque étudiant doit superviser au minimum :

- son routeur `R_XY`,
- son switch `SW_XY`,
- sa VM `dnsdhcp_XY`,
- sa VM `ftpldap_XY`,
- sa VM `client_XY`.

Convention recommandée dans Zabbix :

- **Host group** : `Groupe_X_Etudiant_Y`
- **Hosts** : `R_XY`, `SW_XY`, `dnsdhcp_XY`, `ftpldap_XY`, `client_XY`
- **Tags** : `groupe=X`, `etudiant=Y`, `role=routeur|switch|vm`

Templates conseillés :

- `Linux by Zabbix agent` pour les VMs Ubuntu 24.04.
- SNMP template Cisco pour routeur/switch (si SNMP activé sur IOS).
- `ICMP Ping` minimum sur tous les équipements.

#### IV.6.6 Agent Zabbix sur Ubuntu 24.04

Sur chaque VM Ubuntu supervisée :

```bash
sudo apt update
sudo apt install -y zabbix-agent2
sudo sed -i "s/^Server=.*/Server=10.X.Y.2/" /etc/zabbix/zabbix_agent2.conf
sudo sed -i "s/^ServerActive=.*/ServerActive=10.X.Y.2/" /etc/zabbix/zabbix_agent2.conf
sudo sed -i "s/^Hostname=.*/Hostname=client_XY/" /etc/zabbix/zabbix_agent2.conf
sudo systemctl enable --now zabbix-agent2
```

Répéter en ajustant `Hostname` (`dnsdhcp_XY`, `ftpldap_XY`, etc.).

#### IV.6.7 Supervision Cisco par SNMP (minimum)

Sur IOS (routeur/switch), exemple simple :

```plaintext
conf t
snmp-server community TPpublic RO
snmp-server location Salle_X
snmp-server contact enseignant@lab.local
end
write memory
```

Dans Zabbix :

1. Créer l’hôte (`R_XY` ou `SW_XY`) avec IP de management.
2. Ajouter interface SNMP v2, communauté `TPpublic`.
3. Lier un template SNMP Cisco.
4. Vérifier la collecte (dernières données + graphes).

---

## V. Procédure de test et validation

### V.1 Vérifications sur les équipements Cisco

- **Switch 2960** :
  - `show vlan brief` : VLAN 10 et 20 présents, ports associés.
  - `show interfaces trunk` : port vers R_XY en trunk, VLANs autorisés corrects.
  - `show interfaces status` : vérifier que les ports sont `connected` et non `err-disabled`.

- **Routeur 2800 (R_XY)** :
  - `show ip interface brief` : sous-interfaces `G0/0.10`, `G0/0.20` et interface `G0/1` up avec les bonnes IP.
  - `show ip route` : routes OSPF présentes vers d’autres LAN Data/Mgmt.
  - `show ip ospf neighbor` : au moins un voisin OSPF (le Core, les autres routeurs).

- **ASA 5512-X** :
  - `show interface ip brief` : inside / outside up avec bonnes IP.
  - `show route` : route par défaut vers ISP présente.
  - `show nat`, `show xlate` : vérifier la présence de traductions lors d’un trafic vers Internet.

---

### V.2 Tests de connectivité IP

1. **Intra-VLAN Data (même X, Y différents)**
   - Depuis `PC_XY`, ping un autre PC Data du même groupe (si disponible).
   - Résultat attendu : **Ping OK**.

2. **Inter-VLAN via routage**
   - Depuis `PC_XY` (VLAN 10), ping une IP de Management (VLAN 20) autorisée (par ex. SVI du switch ou serveur).
   - Résultat attendu : **Ping OK**, tracé montrant passage par `R_XY`.

3. **Accès à Internet via ASA**
   - Depuis `PC_XY`, ping une IP publique autorisée (ex. `8.8.8.8` ou cible labo).
   - Résultat attendu : **Ping OK**, NAT visible sur ASA (`show xlate`).

4. **Routage inter-élèves (optionnel)**
   - Tester un ping / traceroute d’un PC d’un étudiant à un autre PC d’étudiant (autre Y, même X ou autre X) si la politique de routage le permet.

---

### V.3 Tests des services

- **DHCP (Kea)** :
  - Sur `client_XY`, forcer un renouvellement DHCP et observer l’IP, la passerelle et le DNS obtenus.
  - Sur le serveur, vérifier les logs Kea (fichier de leases ou `journalctl`) pour voir la requête.

- **DNS (Bind9)** :
  - Depuis `PC_XY` ou la VM, lancer :
    - `dig srv.x.lab.local`
    - `dig core.x.lab.local`
    - `dig -x 10.X.Y.1`
  - Vérifier que les réponses correspondent à votre plan IP.

- **LLDAP** :
  - Accéder à l’interface web LDAP.
  - Tester la connexion avec un compte admin, lister les utilisateurs, vérifier la présence des OU et des comptes de test.

- **ProFTPD** :
  - Depuis `PC_XY`, se connecter en FTP (ou FTPS) avec un compte LDAP.
  - Effectuer un upload / download de fichier et vérifier les logs ProFTPD.

- **Zabbix 7.4.6** :
  - Vérifier l’accès web à `http://10.X.Y.2/zabbix`.
  - Vérifier que les 5 hôtes minimum (`R_XY`, `SW_XY`, `dnsdhcp_XY`, `ftpldap_XY`, `client_XY`) sont en état "Enabled/Available".
  - Produire une capture de l’écran "Latest data" et au moins un graphe (CPU VM ou interface réseau Cisco).

---

## VI. Grille d’évaluation (barème sur 20)

> **Objectif** : disposer de critères clairs, techniques, et vérifiables.
> Les points peuvent être ajustés par l’enseignant selon la durée et la profondeur souhaitées.

### 1. Topologie & adressage (4 pts)

- **(2 pts)** : Plan d’adressage complet, cohérent, respectant les variables \(X\) et \(Y\) (Data, Mgmt, Transit).
- **(2 pts)** : Documentation des schémas (logique + physique) et correspondances interfaces/ports.

### 2. Configuration L2/L3 Cisco & OSPF (6 pts)

- **(2 pts)** : VLANs, ports access, trunks correctement configurés (2960).
- **(2 pts)** : Sous-interfaces dot1Q et IP correctement configurées (2800).
- **(2 pts)** : OSPF fonctionnel (adjacences stables, routes correctes).

### 3. ASA & sécurité (4 pts)

- **(2 pts)** : Interfaces ASA correctly configurées (inside/outside, IP, security-level, routes).
- **(2 pts)** : NAT fonctionnel permettant un accès "Internet" depuis les LANs internes.

### 4. Services Kea / Bind9 / LLDAP / ProFTPD / Zabbix (6 pts)

- **(2 pts)** : Kea & Bind9 opérationnels (clients obtiennent IP + résolution de noms OK).
- **(2 pts)** : LLDAP & ProFTPD intégrés (auth LDAP fonctionnelle, tests FTP concluants).
- **(2 pts)** : Zabbix opérationnel (appliance déployée, hôtes de l’étudiant supervisés, métriques visibles).

### Pénalités / bonus

- **Bonus possibles** :
  - +1 pt pour documentation particulièrement soignée (diagrammes, runbook).
  - +1 pt pour mise en place d’éléments supplémentaires justifiés (Routage inter entreprises).

---


Forcer l'utilisation de la mac pour l'id dhcp.
```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```
Ajoutez la dernière ligne
```bash
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp-identifier: mac  # <-- Ajoute cette ligne ici
```
Recharger netplan

```bash
sudo netplan apply
```
### Ce qui est attendu dans le rapport

- Fiche de **plan d’adressage** (tableaux complétés pour vos valeurs \(X\) et \(Y\)).
- Extraits de **configurations Cisco/ASA**, Kea, Bind9, LLDAP, ProFTPD, Zabbix (commentés).
- **Recette de tests** (liste de commandes + résultats) et conclusions.

