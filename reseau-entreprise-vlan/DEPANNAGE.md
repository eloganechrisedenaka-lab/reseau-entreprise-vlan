# Dépannage : pannes simulées

Méthode appliquée, couche par couche : interface et câblage, VLAN et trunk, adressage (`ipconfig`), passerelle (`ping`), routage (`show ip route`), NAT et ACL. Une seule panne à la fois, état initial sauvegardé avant chaque test.

---

## Panne n°1 : sous-interface VLAN 20 sans encapsulation 802.1Q

**Introduite par :** `interface g0/0.20` puis `no encapsulation dot1q 20`

**Symptôme :** les PC de la comptabilité n'ont plus accès à Internet ni au serveur. Les VLAN 10 et 30 fonctionnent.

**Diagnostic :**
1. `ipconfig /renew` sur PC-COMPTA1 : le DHCP échoue.
2. `ping 192.168.20.1` : [À COMPLÉTER avec le résultat observé : échec attendu, la passerelle ne reçoit plus les trames taguées VLAN 20].
3. `show ip interface brief` sur R1 : `g0/0.20` reste up/up, rien d'anormal.
4. `show running-config interface g0/0.20` : la ligne `encapsulation dot1q 20` est absente.

**Cause :** sans encapsulation, R1 ne traite plus les trames taguées VLAN 20. Ni le DHCP, ni la passerelle, ni le routage ne fonctionnent pour ce VLAN. L'état up/up ne reflète que les couches 1 et 2 de l'interface, pas la justesse de sa configuration logique.

**Correction :**
```
interface g0/0.20
 encapsulation dot1q 20
```

**Vérification :** `ipconfig /renew` sur PC-COMPTA1 (IP 192.168.20.x), `ping 192.168.20.1`, `ping 8.8.8.8`.

---

## Panne n°2 : port d'accès affecté au mauvais VLAN

**Introduite par :** `interface fa0/3` puis `switchport access vlan 30`

**Symptôme :** PC-COMPTA1 a Internet mais n'atteint plus les PC de l'administration ni PC-COMPTA2.

**Diagnostic :**
1. `ipconfig /renew` : le DHCP répond, mais l'IP reçue (192.168.30.13) appartient au sous-réseau des invités.
2. `show vlan brief` sur SW1 : Fa0/3 est dans le VLAN 30 (INVITES).
3. `show access-lists` sur R1 : les compteurs des lignes `deny` augmentent quand PC-COMPTA1 teste PC-ADM1.

**Cause :** le port est dans le VLAN des invités. Le PC reçoit une adresse invité et subit l'ACL `BLOCK-INVITES`, qui bloque les accès vers les VLAN 10 et 20 mais autorise Internet (`permit ip any any`). Panne silencieuse : aucune erreur visible, et dans un cas inverse un PC placé par erreur dans un VLAN privilégié obtiendrait des accès non prévus.

**Correction :**
```
interface fa0/3
 switchport access vlan 20
```

**Vérification :** `ipconfig /renew` (IP 192.168.20.x), `ping 192.168.20.12`, `ping 192.168.10.11`, `ping 8.8.8.8` depuis PC-COMPTA1. Le bail 192.168.30.13 peut être supprimé sur R1 avec `clear ip dhcp binding 192.168.30.13`.

---

## Panne n°3 : VLAN Administration sans accès Internet (NAT inside retiré)

**Introduite par :** `interface g0/0.10` puis `no ip nat inside`

**Symptôme :** PC-ADM1 joint la comptabilité et ses collègues, mais pas Internet (`ping 8.8.8.8` en échec).

**Diagnostic :**
1. `ping 8.8.8.8` depuis PC-ADM1 : échec.
2. `ping 192.168.20.11` et `.12` : réussi, le problème est limité à la sortie vers l'extérieur.
3. `show ip nat translations` : aucune traduction pour 192.168.10.x, alors que le VLAN 20 en a.
4. `show ip nat statistics` : `g0/0.10` absente des interfaces inside.
5. `show running-config interface g0/0.10` : `ip nat inside` absent.

**Cause :** `g0/0.10` n'est plus déclarée interface interne du NAT. Les paquets du VLAN 10 sont routés sans traduction : ils sortent avec leur adresse privée, et R-ISP, qui n'a pas de route vers ce réseau, ne peut pas renvoyer les réponses. Le routage inter-VLAN ne passe pas par le NAT, ce qui explique que les communications internes continuent de fonctionner.

**Correction :**
```
interface g0/0.10
 ip nat inside
exit
clear ip nat translation *
```

**Vérification :** `ping 8.8.8.8` depuis PC-ADM1 réussi ; `show ip nat translations` montre des lignes 192.168.10.x traduites en 203.0.113.2 (R-ISP ne voit jamais les adresses privées).
