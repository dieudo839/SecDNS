# SecDNS
SecDNS est un projet complet de mise en place d’un serveur DNS sécurisé avec DNSSEC  sur Ubuntu 22.04.  Les objectif etaient de maîtriser Bind9 en mode maître , activer DNSSEC manuellement (dnssec-signzone)  , vérifier la validation avec dig +dnssec , automatiser la configuration (copier-coller)
ta grosse tête

secure-dns-dnssec-ubuntu/README.md

---

````markdown
#  Mise en place d’un service DNS sécurisé (DNSSEC) sur Ubuntu Server 22.04 LTS

##  Description du projet
Ce projet consiste à installer et configurer un **serveur DNS sécurisé avec DNSSEC (Domain Name System Security Extensions)** sur **Ubuntu Server 22.04 LTS**.  
L’objectif est de protéger la résolution de noms contre les attaques telles que le **DNS Spoofing** et le **Cache Poisoning**, en assurant l’intégrité et l’authenticité des réponses DNS.



Objectifs pédagogiques
- Comprendre le fonctionnement d’un serveur DNS (BIND9)
- Mettre en place une zone DNS locale
- Signer cette zone avec DNSSEC
- Vérifier la validation cryptographique des enregistrements DNS
- Documenter la démarche technique pour la réutiliser dans d’autres environnements



Prérequis
- Ubuntu Server 22.04 LTS (machine virtuelle ou physique)
- Accès `sudo`
- Connexion réseau fonctionnelle
- Un client DNS (Ubuntu Desktop, Windows, etc.) pour les tests


**Exemple d’environnement de test (VirtualBox)**

| Machine | Rôle | Adresse IP | Nom de domaine |
|----------|------|-------------|----------------|
| Ubuntu Server | Serveur DNS | 192.168.56.10 | ns1.arc.local |
| Client | Machine de test | 192.168.56.20 | - |

---

##  Étape 1 – Installation du service DNS (BIND9)
```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc dnsutils -y
sudo systemctl status bind9
````

> ✅ Le service doit être **active (running)**.

---

##  Étape 2 – Configuration d’une zone DNS locale

### 1️⃣ Déclaration de la zone

```bash
sudo nano /etc/bind/named.conf.local
```

Ajouter :

```bash
zone "arc.local" {
    type master;
    file "/etc/bind/db.arc.local";
};
```

### 2️⃣ Création du fichier de zone

```bash
sudo cp /etc/bind/db.local /etc/bind/db.arc.local
sudo nano /etc/bind/db.arc.local
```

Contenu du fichier :

```bash
$TTL    604800
@       IN      SOA     ns1.arc.local. admin.arc.local. (
                             2         ; Serial
                            604800     ; Refresh
                             86400     ; Retry
                           2419200     ; Expire
                            604800 )   ; Negative Cache TTL
;
@       IN      NS      ns1.arc.local.
ns1     IN      A       10.0.2.15
www     IN      A       10.0.2.25
```

### 3️⃣ Redémarrage du service

```bash
sudo systemctl restart bind9
```

---

##  Étape 3 – Test de la résolution DNS

Depuis le serveur ou une autre VM :

```bash
dig @10.0.2.15 www.arc.local
```

> ✅ Le domaine doit se résoudre avec l’adresse IP configurée.

---

##  Étape 4 – Activation de DNSSEC

### 1️⃣ Génération des clés

```bash
cd /etc/bind
sudo dnssec-keygen -a RSASHA256 -b 2048 -n ZONE arc.local
sudo dnssec-keygen -f KSK -a RSASHA256 -b 4096 -n ZONE arc.local
```

Deux fichiers de clés seront créés :

```
Karc.local.+008+59443.key
Karc.local.+008+32089.key
```

### 2️⃣ Inclusion des clés dans la zone

```bash
sudo nano /etc/bind/db.arc.local
```

Ajouter :

```bash
$INCLUDE "Karc.local.+008+59443.key"
$INCLUDE "Karc.local.+008+32089.key"
```

### 3️⃣ Signature de la zone

```bash
sudo dnssec-signzone -A -3 random -N increment -o arc.local -t db.arc.local
```

### 4️⃣ Mise à jour de la configuration

```bash
sudo nano /etc/bind/named.conf.local
```

Modifier :

```bash
zone "arc.local" {
    type master;
    file "/etc/bind/db.arc.local.signed";
};
```

### 5️⃣ Redémarrage de BIND

```bash
sudo systemctl restart bind9
```

---

## 🔎 Étape 5 – Vérification de DNSSEC

Test :

```bash
dig +dnssec www.arc.local
```

> ✅ Si la ligne `flags: qr rd ra ad` apparaît, la signature DNSSEC est valide.

---

##  Étape 6 – Résultats attendus

| Étape        | Commande                           | Résultat attendu        |
| ------------ | ---------------------------------- | ----------------------- |
| Installation | `systemctl status bind9`           | Service actif           |
| Test DNS     | `dig @10.0.2.15 www.arc.local` | Résolution correcte     |
| Test DNSSEC  | `dig +dnssec www.arc.local`        | Flag **ad** présent     |
| Signature    | `dnssec-signzone`                  | Zone signée avec succès |

---

##  Explications théoriques

* **BIND9** : le logiciel standard pour implémenter un serveur DNS.
* **DNSSEC** : ajoute une signature cryptographique aux enregistrements DNS.
* **KSK (Key Signing Key)** : signe la clé ZSK.
* **ZSK (Zone Signing Key)** : signe les enregistrements DNS.
* **But** : garantir que les réponses DNS proviennent bien de la source légitime et n’ont pas été altérées.

---

##  Arborescence du projet

```
/etc/bind/
├── db.arc.local
├── db.arc.local.signed
├── Karc.local.+008+12345.key
├── Karc.local.+008+67890.key
└── named.conf.local
```

---

##  Commandes utiles

| Commande                 | Description                |
| ------------------------ | -------------------------- |
| `systemctl status bind9` | Vérifie l’état du service  |
| `sudo rndc reload`       | Recharge la configuration  |
| `dig +dnssec`            | Teste la résolution DNSSEC |
| `dnssec-keygen`          | Génère une clé DNSSEC      |
| `dnssec-signzone`        | Signe une zone DNS         |

---

Améliorations possibles

* Ajouter un serveur DNS secondaire pour la redondance
* Configurer un serveur de validation DNSSEC côté client
* Intégrer le DNS sécurisé à un service Web HTTPS


  Auteur

Arcélia Banhoro
Étudiante en Informatique – ESI
Projet GitHub : secure-dns-dnssec-ubuntu

