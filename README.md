# Orbitale

API de billetterie (concerts, spectacles) - projet du module Web services, M2 YNOV.

## Contrat

Le contrat de l'API est dans `openapi.yaml` (OpenAPI 3.1).

Vérifier le contrat :

```
npx --yes @stoplight/spectral-cli@6 lint openapi.yaml
```

Lancer le mock sur le port 4010 :

```
npx --yes @stoplight/prism-cli@5 mock openapi.yaml -p 4010
```

## Docs

- `docs/notes_s1.md` : réponses du TP guidé S1
- `docs/ressources.md` : ressources, relations et endpoints
- `docs/mock.png` : tests du mock Prism
- `docs/swagger.png` : le contrat dans Swagger UI
