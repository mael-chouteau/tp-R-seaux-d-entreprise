# Checklist d’évaluation — TP Conception et urbanisation des services réseau

Sur **Ubuntu Server 24.04** (VMs client_XY, dnsdhcp_XY, ftpldap_XY), installez les outils suivants pour les tests :

```bash
sudo apt update
sudo apt install -y dnsutils ldap-utils traceroute lftp curl
```

| Paquet      | Outils fournis                    | Usage                                      |
|-------------|-----------------------------------|--------------------------------------------|
| `dnsutils`  | `dig`, `nslookup`                 | Tests DNS (Bind9)                          |
| `ldap-utils`| `ldapsearch`, `ldapwhoami`        | Tests LDAP (LLDAP)                         |
| `traceroute`| `traceroute`                      | Traceroute (passage par R_XY)              |
| `lftp`      | `lftp`                            | Client FTP/FTPS (ProFTPD)                  |
| `curl`      | `curl`                            | Tests HTTP, vérification de services       |

> `dhclient` (DHCP) et `ip`/`ss` (réseau) sont déjà présents par défaut.

---

## 1. Topologie & adressage



### Plan d’adressage

- [ ] Tableau d’adressage complété avec vos valeurs X et Y
- [ ] LAN Data (VLAN 10) : `10.X.Y.0/24`, passerelle `10.X.Y.254`
- [ ] LAN Mgmt (VLAN 20) : `10.X.2Y.0/24` (2Y = 20+Y), passerelle `10.X.2Y.254`
- [ ] Transit : `172.16.X.0/24`
- [ ] Adresses des VMs cohérentes (dnsdhcp `10.X.Y.1`, zabbix `10.X.Y.2`, ftpldap `10.X.Y.3`, client `10.X.Y.4`)
- [ ] PC Management : `10.X.2Y.100`, passerelle `10.X.2Y.254`

---

## 2. Switch Cisco 2960 (SW_XY)

### VLANs

- [ ] `show vlan brief` : VLAN 10 (DATA_XY) et 20 (MGMT_X) présents
- [ ] Ports correctement associés aux VLANs

### Port Fa0/1 (vers PC + hyperviseur)

- [ ] Mode trunk (pas access)
- [ ] VLAN natif 20
- [ ] VLANs autorisés : 10, 20
- [ ] `show interfaces trunk` : Fa0/1 affiche Native 20, Allowed 10,20

### Port Fa0/24 (vers routeur)

- [ ] Mode trunk
- [ ] VLANs autorisés : 10, 20

### SVI Management

- [ ] Interface Vlan20 : IP `10.X.2Y.253/24`
- [ ] `ip default-gateway 10.X.2Y.254`
- [ ] `no shutdown` sur Vlan20

### État

- [ ] `show interfaces status` : ports `connected`, pas `err-disabled`

---

## 3. Routeur Cisco 2800 (R_XY)

### Sous-interfaces

- [ ] `show ip interface brief` : Fa0/0.10 et Fa0/0.20 up (ou G0/0 selon modèle)
- [ ] Sous-if Data : IP `10.X.Y.254`
- [ ] Sous-if Mgmt : IP `10.X.2Y.254`
- [ ] Encapsulation dot1Q correcte (10 et 20)

### Interface Transit

- [ ] Interface Transit : IP `172.16.X.Y/24` (ou convention du groupe)

### OSPF

- [ ] `show ip ospf neighbor` : au moins un voisin (Core ou autre R_XY)
- [ ] `show ip route ospf` : routes vers LAN Data et Transit
- [ ] Réseaux annoncés : `10.X.Y.0` (Data), `172.16.X.0` (Transit) — **pas** `10.X.2Y.0` (Mgmt reste interne)

### ACL — Isolation VLANs et confinement Mgmt

- [ ] VLAN 10 ↔ VLAN 20 : pas de communication (ping bloqué)
- [ ] VLAN 20 : ne sort pas vers le transit (`show ip access-lists`)
- [ ] VLAN 10 : peut atteindre le transit et les autres élèves

---

## 4. ASA 5512-X

### Interfaces

- [ ] `show interface ip brief` : inside et outside up
- [ ] Inside : IP correcte (ex. `192.168.X.2/30`)
- [ ] Outside : IP fournie par l’ISP

### Route par défaut

- [ ] `show route` : route `0.0.0.0` vers passerelle ISP

### NAT

- [ ] NAT configuré (inside → outside)
- [ ] `show xlate` : traductions visibles lors d’un trafic vers Internet

---

## 5. Tests de connectivité IP

### Intra-VLAN Management

- [ ] Depuis PC_XY (VLAN 20), ping `10.X.2Y.253` (SVI switch) → OK

### Isolation VLAN 10 / VLAN 20 (ACL)

- [ ] Depuis PC_XY (VLAN 20), ping `10.X.Y.1` (VLAN 10) → **doit échouer** (ACL)
- [ ] Depuis client_XY (VLAN 10), ping `10.X.2Y.253` (SVI Mgmt) → **doit échouer** (ACL)

### VLAN 10 vers transit / autres élèves

- [ ] Depuis client_XY (VLAN 10), ping une VM d’un autre élève ou le Core → OK
- [ ] Traceroute montre passage par R_XY

```bash
# Depuis client_XY — traceroute vers une VM Data d’un autre élève (même groupe)
traceroute 10.X.Yprime.1
traceroute 172.16.X.254
```

### Internet

- [ ] Depuis client_XY (VLAN 10), ping `8.8.8.8` (ou cible labo) → OK  
  > Le PC_XY (VLAN 20) n’a pas accès Internet (Mgmt interne).

```bash
# Depuis client_XY — vérifier la connectivité Internet
traceroute 8.8.8.8
curl -s -o /dev/null -w "%{http_code}" https://www.google.com
```

---

## 6. VMs et réseau

### Netplan VLAN 10 (si applicable)

- [ ] Interface VLAN 10 configurée (ex. `enp0s3.10`)
- [ ] Adresses statiques correctes pour dnsdhcp, ftpldap, zabbix
- [ ] client_XY : DHCP sur VLAN 10 avec `dhcp-identifier: mac`

```bash
# Vérifier la config réseau (VLAN, IP, routes)
ip addr show
ip route show
ip link show
```

### Kea DHCP

- [ ] Scope `10.X.Y.0/24` (réseau Data)

```bash
# Sur dnsdhcp_XY — vérifier que le scope correspond au VLAN 10
grep -A2 '"subnet4"' /etc/kea/kea-dhcp4.conf
grep 'interfaces-config' -A2 /etc/kea/kea-dhcp4.conf
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

---

## 7. Services

### Kea DHCP

- [ ] `client_XY` obtient une IP en `10.X.Y.10`–`10.X.Y.99`
- [ ] Passerelle `10.X.Y.254`, DNS `10.X.Y.1` reçus
- [ ] `journalctl -u kea-dhcp4-server` ou fichier leases : requêtes visibles

```bash
# Sur client_XY — forcer un renouvellement DHCP
sudo dhclient -r enp0s3.10    # ou eth0.10 selon l'interface
sudo dhclient enp0s3.10

# Vérifier l’IP, la passerelle et le DNS obtenus
ip addr show enp0s3.10
ip route show
resolvectl status

# Sur dnsdhcp_XY — vérifier les baux et les logs
sudo cat /var/lib/kea/kea-leases4.csv
sudo journalctl -u kea-dhcp4-server -n 20 --no-pager
```

### Bind9 DNS

- [ ] `dig srv.x.lab.local` → `10.X.Y.1`
- [ ] `dig core.x.lab.local` → IP Core
- [ ] `dig -x 10.X.Y.1` → résolution inverse OK

```bash
# Depuis client_XY ou dnsdhcp_XY — tests DNS (adapter x.lab.local à votre groupe)
dig @10.X.Y.1 srv.x.lab.local
dig @10.X.Y.1 core.x.lab.local
dig @10.X.Y.1 -x 10.X.Y.1
dig @10.X.Y.1 ftpldap_XY.x.lab.local

# Sur dnsdhcp_XY — vérifier la syntaxe des zones (adapter x à votre groupe)
sudo named-checkzone x.lab.local /etc/bind/db.x.lab.local
sudo named-checkconf
```

### LLDAP

- [ ] Interface web accessible : `http://10.X.Y.3:17170`
- [ ] Connexion admin OK
- [ ] OU `people` et `groups` présents
- [ ] Comptes de test créés

```bash
# Depuis client_XY ou ftpldap_XY — vérifier que LLDAP répond
ldapsearch -x -H ldap://10.X.Y.3:3890 -b "dc=x,dc=lab,dc=local" -D "uid=admin,ou=people,dc=x,dc=lab,dc=local" -w "MotDePasseAdminLDAP" "(objectClass=*)" dn

# Vérifier les ports LLDAP
ss -tulnp | grep -E '3890|17170'

# Test interface web (doit retourner du HTML)
curl -s -o /dev/null -w "%{http_code}" http://10.X.Y.3:17170
```

### ProFTPD

- [ ] Connexion FTP avec compte LDAP depuis PC_XY ou client_XY → OK
- [ ] Upload / download OK
- [ ] Logs ProFTPD sans erreur

```bash
# Depuis client_XY — connexion FTP avec compte LDAP
lftp -u etudiantXY,MotDePasseLDAP ftp://10.X.Y.3

# Dans lftp :
# ls
# put /tmp/fichier_test.txt
# get fichier_test.txt
# quit

# Alternative en une ligne (upload)
echo "test" > /tmp/test_ftp.txt
lftp -u etudiantXY,MotDePasseLDAP ftp://10.X.Y.3 -e "put /tmp/test_ftp.txt; quit"

# Sur ftpldap_XY — vérifier les logs ProFTPD
sudo journalctl -u proftpd -n 30 --no-pager
```

### Zabbix

- [ ] Accès web : `http://10.X.Y.2/zabbix`
- [ ] 5 hôtes minimum : R_XY, SW_XY, dnsdhcp_XY, ftpldap_XY, client_XY
- [ ] Tous en état "Enabled/Available"
- [ ] Capture "Latest data" + au moins un graphe prêt

```bash
# Vérifier que Zabbix web répond (depuis client_XY)
curl -s -o /dev/null -w "%{http_code}" http://10.X.Y.2/zabbix

# Sur les VMs supervisées — vérifier l’agent Zabbix
sudo systemctl status zabbix-agent2
ss -tulnp | grep zabbix
```

---

## 8. Supervision Cisco (SNMP)

- [ ] SNMP configuré sur routeur et switch : `snmp-server community TPpublic RO`
- [ ] Hôtes Zabbix créés avec interface SNMP

---

## 9. Commandes de vérification rapide

| Équipement | Commande                                                                            |
| ---------- | ----------------------------------------------------------------------------------- |
| Switch     | `show vlan brief` \| `show interfaces trunk` \| `show interfaces status`            |
| Routeur    | `show ip interface brief` \| `show ip ospf neighbor` \| `show ip route`             |
| ASA        | `show interface ip brief` \| `show route` \| `show xlate`                           |
| Kea        | `sudo systemctl status kea-dhcp4-server` \| `sudo cat /var/lib/kea/kea-leases4.csv` |
| Bind9      | `dig @10.X.Y.1 srv.x.lab.local`                                                     |
| LLDAP      | `ss -tulnp \| grep lldap` (ports 3890, 17170)                                       |
| ProFTPD    | `sudo systemctl status proftpd`                                                     |
| Zabbix     | `http://10.X.Y.2/zabbix`                                                            |
