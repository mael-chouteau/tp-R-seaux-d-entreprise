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

- 1 **PC étudiant** connecté à 1 **switch 2960**.
- 1 **routeur 2800** relié au switch par un lien **trunk 802.1Q**.
- Les VLANs principaux sont :
  - **VLAN 10** : Data (poste de travail).
  - **VLAN 20** : Management (accès aux équipements, services, supervision).

Chaque groupe dispose en plus d’un ensemble minimal de **VMs de services** :

- 1 **VM DNS/DHCP** (Kea + Bind9).
- 1 **VM supervision** (Zabbix serveur).
- 1 **VM FTP/LDAP** (LLDAP + ProFTPD).
- 1 **VM client** Linux, utilisée comme poste de test supplémentaire (`client_XY`).

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
- **LAN Management (VLAN 20)** : `10.X.20.0/24`
  - Passerelle par défaut (sous-interface routeur) : `10.X.20.254`
- **Transit inter-routeurs (X)** : `172.16.X.0/24`
  - Adresses précises à définir par groupe pour chaque lien (ex. `.254` pour Core, `.Y` pour chaque `R_XY`, etc.).

### Tableau d’adressage — Modèle générique

#### Rappels de variables

- \(X\) : rangée / groupe (1, 2, 3, …)  
- \(Y\) : étudiant (1, 2, 3, …)

#### Tableau principal

| Rôle / Équipement              | VLAN | Réseau / Masque     | Adresse IP (exemple)         | Passerelle      | Interface / Port               | Commentaire                                      |
|--------------------------------|------|----------------------|------------------------------|-----------------|--------------------------------|-------------------------------------------------|
| PC Data étudiant `PC_XY`       | 10   | 10.X.Y.0/24          | 10.X.Y.100                    | 10.X.Y.254      | `SW_XY` Fa0/1 (access)         | Adresse fournie par DHCP Kea (scope Data)       |
| SW_XY (SVI Mgmt)               | 10   | 10.X.Y.0/24          | 10.X.Y.253                   | 10.X.Y.254      | Vlan 10                        | IP de management du switch                      |
| R_XY – sous-if Data            | 10   | 10.X.Y.0/24          | 10.X.Y.254                   | —               | `G0/0.X` (dot1Q 10)            | Passerelle Data étudiante                       |
| R_XY – sous-if Mgmt            | 20   | 10.X.Y.0/24         | 10.X.Y.252                  | —               | `G0/0.20` (dot1Q 20)           | Passerelle Management                           |
| R_XY – interface Transit       | —    | 172.16.X.0/24        | 172.16.X.Y              | —               | `G0/1`                         | Lien vers switch de transit                     |
| Routeur Core – vers transit    | —    | 172.16.X.0/24        | 172.16.X.254                   | —               | `G0/0`                         | Routeur de bordure, OSPF area 0                 |
| ASA Inside                     | —    | 172.16.X.0/24        | 172.16.X.254                 | —               | `inside`                       | Interface inside, reliée au routeur Core        |
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

- Interface inside : connectée au routeur Core, réseau `172.16.X.0/24`, IP ASA inside `172.16.X.254`.
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
 ip address 172.16.X.254 255.255.255.0
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

### IV.1 Plan générale

- Système : Ubuntu (ou distribution équivalente).
- **VM DNS/DHCP** : `dnsdhcp-xY`, IP `10.X.Y.1` (LAN Data).
- **VM supervision (Zabbix)** : `zabbix-xY`, IP `10.X.Y.2`.
- **VM FTP/LDAP** : `ftp-ldap-xY`, IP `10.X.Y.3`.
- **VM client** : `client_XY`, IP en DHCP dans le scope Data (par exemple `10.X.Y.4`).
- Passerelle par défaut des VMs : `10.X.Y.254`.
- Noms de domaine : `x.lab.local` (à adapter à votre groupe).
---

### IV.2 Kea DHCP (JSON)

Installation (exemple Ubuntu) :

```bash
sudo apt-get update
sudo apt-get install -y kea-dhcp4-server
```

Fichier principal (souvent `/etc/kea/kea-dhcp4.conf`) — exemple simplifié :

```json
{
  "interfaces-config": {
    "interfaces": ["eth0"]
  },
  "lease-database": {
    "type": "memfile",
    "lfc-interval": 3600
  },
  "subnet4": [
    {
      "subnet": "10.X.Y.0/24",
      "pools": [
        { "pool": "10.X.Y.10 - 10.X.Y.99" }
      ],
      "option-data": [
        { "name": "router", "data": "10.X.Y.254" },
        { "name": "domain-name-servers", "data": "10.X.Y.1" },
        { "name": "domain-name", "data": "x.lab.local" }
      ]
    }
  ]
}
```

Redémarrage et statut :

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

> **Couplage avec le réseau** : si Kea n’est pas directement dans les VLANs, configurez un **IP helper** sur le routeur (`ip helper-address <IP_KEA>`) sur les interfaces Data/Mgmt.

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

Installation de base :

```bash
sudo apt-get install -y proftpd-basic proftpd-mod-ldap
```

Configuration ProFTPD (fichier principal ou fragment dédié), points clés :

- Activation du module LDAP.
- Paramètres de connexion à LLDAP : URL, base DN, bind DN + mot de passe.
- Mapping UID/GID à partir d’attributs LDAP (ex. `uidNumber`, `gidNumber`).

Exemple (extrait conceptuel) :

```plaintext
<IfModule mod_ldap.c>
  LDAPServer "ldap://10.X.Y.3"
  LDAPBindDN "cn=admin,dc=x,dc=lab,dc=local" "motdepasse"
  LDAPUsers "ou=users,dc=x,dc=lab,dc=local" "uid=%u"
</IfModule>
```

Tests :

- Connexion FTP / FTPS depuis `PC_XY` avec un compte LDAP.
- Upload / download d’un fichier dans le home de l’utilisateur.

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

### 4. Services Kea / Bind9 / LLDAP / ProFTPD (6 pts)

- **(3 pts)** : Kea & Bind9 opérationnels (clients obtiennent IP + résolution de noms OK).  
- **(3 pts)** : LLDAP & ProFTPD intégrés (auth LDAP fonctionnelle, tests FTP concluants).

### Pénalités / bonus

- **Bonus possibles** :
  - +1 pt pour documentation particulièrement soignée (diagrammes, runbook).
  - +1 pt pour mise en place d’éléments supplémentaires justifiés (Routage inter entreprises).

---

### Ce qui est attendu dans le rapport

- Fiche de **plan d’adressage** (tableaux complétés pour vos valeurs \(X\) et \(Y\)).
- Extraits de **configurations Cisco/ASA**, Kea, Bind9, LLDAP, ProFTPD (commentés).
- **Recette de tests** (liste de commandes + résultats) et conclusions.

