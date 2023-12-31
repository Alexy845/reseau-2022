# TP3 : on va router des trucs

## 1. Echange ARP
🌞Générer des requêtes ARP
<br>marcel 10.3.2.12
<br>john 10.3.2.11
- effectuer un ping d'une machine à l'autre
    - Ping John sur Marcel , ping 10.3.2.12 et les packets sont bien transmits
- observer les tables ARP des deux machines
    - ip neigh show john sur marcel par exemple

    10.3.2.12 dev enp0s8 lladdr 08:00:27:fa:05:2e STALE

    10.3.2.16 dev enp0s8 lladdr 0a:00:27:00:00:12 REACHABLE

- repérer l'adresse MAC de john dans la table ARP de marcel et vice-versa

    - ip neigh show 10.3.2.12 john sur marcel

    10.3.2.12 dev enp0s8 lladdr 08:00:27:fa:05:2e STALE

    - ip neigh show 10.3.2.11 marcel sur john

    10.3.2.11 dev enp0s8 lladdr 08:00:27:8e:02:d1 STALE 

- prouvez que l'info est correcte (que l'adresse MAC que vous voyez dans la table est bien celle de la machine correspondante)
    - et une commande pour afficher la MAC de marcel, depuis marcel

        - ip neigh show 10.3.2.11 marcel sur marcel

            10.3.2.11 dev enp0s8 lladdr 08:00:27:8e:02:d1 STALE

## 2. Analyse de trames

🌞Analyse de trames

- utilisez la commande tcpdump pour réaliser une capture de trame
    -  sudo tcpdump -i enp0s8 -c 10
- videz vos tables ARP, sur les deux machines, puis effectuez un ping
    - sudo ip neigh flush all et ping 10.3.2.12 et ping 10.3.2.12
    
    les packets sont bien transmits 

🦈 Capture réseau tp3_arp.pcapng qui contient un ARP request et un ARP reply

- [Wireshark tp3_arp](./tp3_arp.pcapng.pcap)

# II. Routage

# 1. Mise en place du routage

🌞Activer le routage sur le noeud router
- sudo firewall-cmd --add-masquerade --zone=public --permanent
    - success

🌞Ajouter les routes statiques nécessaires pour que john et marcel puissent se ping

-  sur John , sudo nano /etc/sysconfig/network-scripts/route-enp0s8 , dans le nano 10.3.2.0/24 via 10.3.1.254 dev eth0
    
    GATEWAY= 10.3.1.254
,     
    
    ip route show

    10.3.1.0/24 dev enp0s8 proto kernel scope link src 10.3.1.11 metric 100

    10.3.2.0/24 dev enp0s8 proto kernel scope link src 10.3.2.11 metric 100

- sur Marcel, sudo nano /etc/sysconfig/network-scripts/route-enp0s8 , dans le nano 10.3.1.0/24 via 10.3.2.254 dev eth0

    GATEWAY= 10.3.2.254 ,

    ip route show
    
    10.3.1.0/24 via 10.3.2.254 dev enp0s8 proto static metric 100

    10.3.2.0/24 dev enp0s8 proto kernel scope link src 10.3.2.12 metric 100

- les deux pings fonctionnent , on peut ping john à marcel et inversement les packets sont bien transmits

# 2. Analyse de trames
🌞Analyse des échanges ARP

- videz les tables ARP des trois noeuds
    - sudo ip neigh flush all

| ordre | type trame  | IP source | MAC source              | IP destination | MAC destination            |
|-------|-------------|-----------|-------------------------|----------------|----------------------------|
| 1     | Requête ARP | 10.3.1.11        | John `08:00:27:8e:02:d1` | 10.3.1.254             | Broadcast `00:00:00:00:00` |
| 2     | Réponse ARP | 10.3.1.254         | Marcel  `08:00:27:4b:d9:68`                     | 10.3.1.11              | `John` `08:00:27:8e:02:d1`    |              |                            |
| 3     | Ping        | 10.3.1.11         | John     `08:00:27:8e:02:d1`                    | 10.3.2.12           | Marcel       `08:00:27:4b:d9:68`                     |
| 4     | Pong        |     10.3.2.12    | Marcel  `08:00:27:4b:d9:68`                        | 10.3.1.11             | John     `08:00:27:8e:02:d1`                     |

- [Wireshark tp3_routage_marcel.pcapng](./ping_marcel_vers_john.pcapng) on  retrouve bien les données ICMP

# 3. Accès internet

🌞Donnez un accès internet à vos machines

- sudo nano /etc/sysconfig/network

    GATEWAY= adresse IP de john ou marcel mais on a que le dernier qu'on change x.x.x.254 ex 10.3.2.254 et on redemarre avec 
    
    sudo systemctl restart NetworkManager
    
    ping 8.8.8.8 ça fonctionne on a bien réussi a ping google
- sudo cat /etc/resolv.conf

    ex donné :
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver 1.1.1.1

    et quand on fait dig gitlab.com par ex on nous donne les info de gitlab et perso quand j'ai ping gitlab ou google ou 1.1.1.1 pour testé ça a direct fonctionné 

🌞 Analyse de trames


- effectuer un ping 8.8.8.8 depuis john

    --- 8.8.8.8 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 2016ms
    rtt min/avg/max/mdev = 62.422/125.435/220.156/68.186 ms

- capturez le ping depuis john avec tcpdump 

    sudo tcpdump -i enp0s8 -w john_sur_google.pcap

| ordre | type trame | IP source          | MAC source              | IP destination | MAC destination |     |
|-------|------------|--------------------|-------------------------|----------------|-----------------|-----|
| 1     | ping       | `marcel` `10.3.1.12` | `marcel` `08:00:27:fa:05:2e` | `8.8.8.8`      | google `08:00:27:f5:ef:4b`               |     |
| 2     | pong       | google `8.8.8.8`               | google `08:00:27:f5:ef:4b`                    | `marcel` `10.3.1.12`        | `marcel` `08:00:27:fa:05:2e`    

[tp3_routage_internet.pcapng](./tp3_routage_internet.pcapng)

## 1. Mise en place du serveur DHCP

🌞 Sur la machine john, vous installez et configurez un serveur DHCP (allez Google "rocky linux dhcp server").

- installation du serveur sur john

    sudo dnf install dhcp-server -y
