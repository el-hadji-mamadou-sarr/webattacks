## Exercice 1 : 5Attack

### Ajouter une cible d’attaque à votre environnement : DVWA

```bash
docker run --rm -it -p 8000:80 vulnerables/web-dvwa
```

[image 50-48]

### 2 fonctionnalités de Postman

- Test envoyer une requéte pour soumettre un formulaire pour faire une injectionSQL. J'envoie avec un ID=1

```bash
http://localhost:8000/vulnerabilities/sqli/?id=1&Submit=Submit
```

[image 12-32]

- Test pour faire une injection SQL à l'aveugle

```bash
http://localhost:8000/vulnerabilities/sqli_blind/?id=-1&Submit=Submit
```

image [58-59]

### Test de OWASPZAP

- Lancer OWASP ZAP. J'ai installer OWASP ZAP sur mon système. 

```bash
sudo apt install zaproxy
```

- J'ai ensuite configuré le GUI de OWASP ZAP pour scanner le site DVWA en utilisant l'option "Automated Scan". J'ai entré l'URL de DVWA (http://localhost:8000) et lancé le scan.

image [36-56]

- Après le scan, j'ai examiné les résultats dans l'onglet "Alerts" pour identifier les vulnérabilités potentielles. J'ai trouvé plusieurs alertes.

image [45-33]


### Test NMAP

- J'ai fait un scan de port avec cette commande pour voir les services qui tournent sur le port 8000 

```bash
nmap -sV -p 8000 localhost
```

Les paramètres utilisés sont :
- `-sV` : pour détecter les versions des services en cours d'exécution
- `-p 8000` : pour scanner uniquement le port 8000

images [52-38]

- En scannant les ports du DVWA, j'ai découvert que le serveur est Sur Apache 2.45.25 et potentiellement, je pourrais vérifier s'il y a des vulnérabilités connues pour cette version d'Apache.


- J'ai aussi fait une autre commande pour détecter l'OS qui se trouve dérrière le serveur web

```bash
nmap -A -p 8000 localhost
```

image [58-48]

- Le paramètre `-A` active la détection avancée, y compris la détection du système d'exploitation, des versions, des scripts et du traceroute.

- Là on voit une redirection automatique vers login.php, le cookie PHPSESSID n'a pas le flag HTTP Only ce qui pourrais potentiellement entrainer un vol de cookie de session.


### Test Nikto

- J'ai utilisé Nikto pour scanner le serveur web DVWA à la recherche de vulnérabilités connues. La commande utilisée est la suivante :

```bash
nikto -h http://localhost:8000
```

image [58-48]

- On voit plusieurs vulnérabilités de sécurité importantes:
    - Il y'a des informations qui ont fuités notamment le répertoire /config/
    - encore la page login.php qui est détectée qu'on peut potentiellement brute force pour accéder au DVWA

- j'ai aussi réalisé ce test avec Nikto pour scanner uniquement les vulnérabilités liées à Apache avec cette commande :

```bash
nikto -h http://localhost:8000 -Plugins "apache"
```

image [58-48]

- Dans le réultat le résultat du scan, rien n'a été détecté.

