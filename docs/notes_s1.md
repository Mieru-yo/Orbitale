# Notes TP S1

Toutes les commandes ont été lancées sous PowerShell, d'où `curl.exe`.

## Étape 1 - Anatomie d'un échange

```
curl.exe -v https://api.github.com/repos/nodejs/node -o NUL
```

```
> GET /repos/nodejs/node HTTP/1.1
> Host: api.github.com
> User-Agent: curl/8.21.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json; charset=utf-8
< Cache-Control: public, max-age=60, s-maxage=60
< ETag: W/"a4b8505f4fc2c602c04de51ff79227fd50716f4c9bbd82037d65d54d1e16f618"
< Last-Modified: Fri, 20 Nov 2026 10:55:08 GMT
< X-RateLimit-Limit: 60
< X-RateLimit-Remaining: 55
< X-RateLimit-Reset: 1795177650
< Content-Length: 6618
```

**1. Recopie la ligne de requête et la ligne de statut. Quelle version de HTTP est utilisée ?**

La ligne de requête est `GET /repos/nodejs/node HTTP/1.1` et la ligne de statut est `HTTP/1.1 200 OK`.
La version utilisée est HTTP/1.1.

**2. Relève quatre en-têtes de réponse et explique le rôle de chacun.**

- Content-Type indique que le corps est du JSON.
- ETag donne la version de la ressource.
- Cache-Control indique que la réponse peut être réutilisée pendant 60 secondes.
- Last-Modified donne la date de dernière modification du dépôt.

**3. Que signifie x-ratelimit-remaining ? Que se passera-t-il quand il atteindra 0 ?**

C'est le nombre de requêtes qu'il reste sur les 60 autorisées par heure sans token.
Quand il atteint 0, GitHub refuse les requêtes avec une erreur 403 jusqu'à la fin de l'heure.

## Étape 2 - Cache conditionnel

```
$etag = (curl.exe -sI https://api.github.com/repos/nodejs/node | Select-String '^etag:').Line -replace '^etag:\s*',''
curl.exe -s -o NUL -w "%{http_code}`n" -H ('If-None-Match: ' + ($etag -replace '"','\"')) https://api.github.com/repos/nodejs/node
```

```
304
```

(Sous PowerShell il faut échapper les guillemets de l'ETag, sinon on reçoit 200.)

**1. Quel statut obtiens-tu ? Pourquoi le corps est-il vide ?**

On obtient un 304 Not Modified.
Le corps est vide parce que la ressource n'a pas changé, le client peut garder sa copie.

**2. Quel est l'intérêt pour une application mobile ? Et pour le quota de GitHub ?**

Pour une appli mobile, ça évite de retélécharger les données, donc c'est plus rapide et ça consomme moins de data.
Pour le quota GitHub, un 304 n'est pas décompté si on est authentifié (sans token, il l'est quand même).

## Étape 3 - Les erreurs

```
curl.exe -s -i https://api.github.com/repos/nodejs/ce-depot-n-existe-pas
curl.exe -s -i -X DELETE https://api.github.com/repos/nodejs/node
curl.exe -s -i "https://geo.api.gouv.fr/communes?codePostal=abc"
```

```
HTTP/1.1 404 Not Found
{"message": "Not Found", "documentation_url": "https://docs.github.com/rest/repos/repos#get-a-repository", "status": "404"}

HTTP/1.1 401 Unauthorized
{"message": "Requires authentication", "documentation_url": "https://docs.github.com/rest", "status": "401"}

HTTP/1.1 200 OK
[]
```

**1. Remplis un tableau : API, requête, statut, format du corps d'erreur, information utile pour corriger.**

| API | Requête | Statut | Format du corps | Info utile pour corriger |
|---|---|---|---|---|
| GitHub | GET d'un dépôt inexistant | 404 | JSON message, documentation_url, status | "Not Found" et un lien vers la doc |
| GitHub | DELETE sans token | 401 | même JSON | "Requires authentication" |
| geo.api.gouv.fr | codePostal=abc | 200 | pas d'erreur, liste vide | aucune |

Les trois sont des erreurs client (4xx), il n'y a aucune erreur serveur (5xx).

**2. Une de ces erreurs ne respecte pas la sémantique HTTP : laquelle, et pourquoi ?**

C'est celle de geo.api.gouv.fr.
"abc" n'est pas un code postal valide, l'API devrait renvoyer un 400 et pas un 200.

**3. Réécris l'une de ces erreurs au format RFC 9457 (type, title, status, detail).**

Voici l'erreur de geo.api.gouv.fr réécrite au format RFC 9457 :

```
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "type": "https://geo.api.gouv.fr/problems/parametre-invalide",
  "title": "Paramètre invalide",
  "status": 400,
  "detail": "codePostal doit contenir 5 chiffres, reçu : abc"
}
```

## Étape 4 - Collections et pagination

```
curl.exe -s "https://geo.api.gouv.fr/communes?nom=Lyon&fields=nom,code,population"
curl.exe -sI "https://api.github.com/repos/nodejs/node/issues?per_page=5" | Select-String '^link:'
```

```
[{"nom":"Lyon","code":"69123","population":519127,"_score":0.538}, {"nom":"Cognat-Lyonne","code":"03080","population":706,"_score":0.442}, ...]

Link: <https://api.github.com/repositories/27193779/issues?per_page=5&after=...&page=2>; rel="next"
```

**1. Avec geo.api.gouv.fr, comment filtre-t-on une collection ? Comment choisit-on les champs renvoyés ?**

On filtre avec un paramètre dans l'URL, par exemple `?nom=Lyon`.
On choisit les champs avec le paramètre `fields`, par exemple `?fields=nom,code,population`.

**2. Avec GitHub, où se trouve l'information de page suivante : dans le corps ou dans un en-tête ? Lequel ?**

L'information est dans un en-tête, `Link`, et pas dans le corps.

**3. Quel avantage à mettre les liens de pagination dans un en-tête ?**

L'avantage est que le corps reste une simple liste, que le client peut utiliser directement. Il trouve la page suivante dans l'en-tête, sans que ça se mélange aux données.

## Étape 5 - Négociation de contenu

```
curl.exe -s -i -H "Accept: application/vnd.github.raw+json" https://api.github.com/repos/nodejs/node/readme
curl.exe -s -i -H "Accept: application/vnd.github.html+json" https://api.github.com/repos/nodejs/node/readme
curl.exe -s -i -H "Accept: application/xml" https://api.github.com/repos/nodejs/node/readme
```

```
HTTP/1.1 200 OK
Content-Type: application/vnd.github.raw+json; charset=utf-8
# Node.js
...

HTTP/1.1 200 OK
Content-Type: application/vnd.github.html+json; charset=utf-8
<div id="readme" class="md" data-path="README.md"><article class="markdown-body ...
...

HTTP/1.1 415 Unsupported Media Type
```

**1. Quel en-tête choisit le format ?**

C'est l'en-tête `Accept` qui choisit le format.

**2. Quel statut un serveur devrait-il renvoyer si aucun des formats demandés n'est disponible ?**

Le serveur devrait renvoyer un 406 Not Acceptable.

## Étape 6 - Échelle de Richardson

```
curl.exe -s https://api.github.com/repos/nodejs/node | jq -c '{url, issues_url, owner: .owner.url}'
curl.exe -s -i https://geo.api.gouv.fr/communes/99999
curl.exe -s -o NUL -w "%{http_code}`n" -X POST https://geo.api.gouv.fr/communes
curl.exe -s "https://pokeapi.co/api/v2/pokemon?limit=2"
curl.exe -s -i https://pokeapi.co/api/v2/pokemon/inexistant
```

```
{"url":"https://api.github.com/repos/nodejs/node","issues_url":"https://api.github.com/repos/nodejs/node/issues{/number}","owner":"https://api.github.com/users/nodejs"}

HTTP/1.1 404 Not Found

401

{"count":1351,"next":"https://pokeapi.co/api/v2/pokemon?offset=2&limit=2","previous":null,"results":[{"name":"bulbasaur","url":"https://pokeapi.co/api/v2/pokemon/1/"}, ...]}

HTTP/1.1 404 Not Found
```

**Classe GitHub, geo.api.gouv.fr et une API que tu connais sur l'échelle de Richardson. Justifie chaque classement par une requête observée.**

- GitHub est au niveau 2 : `GET /repos/nodejs/node` renvoie 200 et un dépôt inexistant renvoie 404.
- geo.api.gouv.fr est au niveau 2 : `GET /communes/69123` renvoie 200 et une commune inexistante renvoie 404.
- PokéAPI est au niveau 2 : `GET /api/v2/pokemon/1` renvoie 200 et un pokémon inexistant renvoie 404.
