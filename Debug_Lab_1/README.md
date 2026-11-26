# Debug Lab 1 : le Service qui perd des requêtes

## Étape 1 : déploiement

### Manifest

Le fichier `debug-lab-service.yaml` contient deux Deployments (`api-alpha`, `api-beta`, 2 réplicas chacun, une même image) et un Service `api-service` (`port: 80`, `targetPort: 5678`, selector `app: mon-api`).

### Commandes et résultats

```bash
kubectl apply -f debug-lab-service.yaml
kubectl get pods
```

Résultat : 4 pods en `Running` (2 `api-alpha`, 2 `api-beta`).

Le problème est invisible à ce stade.

## Étape 2 : diagnostic

### Reproduire l'erreur

Je lance un pod temporaire avec `curl` pour envoyer des requêtes depuis l'intérieur du cluster :


```
~ $ curl http://api-service
curl: (7) Failed to connect to api-service:80 after 225 ms: Could not connect to server
~ $ curl http://api-service
Reponse du deploiement Alpha
~ $ curl http://api-service
curl: (7) Failed to connect to api-service:80 after 12 ms: Could not connect to server
~ $ curl http://api-service
Reponse du deploiement Alpha
~ $ curl http://api-service
curl: (7) Failed to connect to api-service:80 after 1 ms: Could not connect to server
```

Analyse :

- Les réponses alternent entre échec et succès : une requête sur deux échoue.
- Seul Alpha répond, jamais Beta.

### Comparer les logs d'Alpha et de Beta

Les deux Deployments utilisent la même image, la différence est donc dans la configuration. Je compare les logs d'un pod Alpha et d'un pod Beta :

```bash
kubectl get pods
kubectl logs api-alpha-fd845d97b-7kc6s
kubectl logs api-beta-5df6b7557d-gfxdh
```

```
# api-alpha
2026/11/23 13:14:44 [INFO] server is listening on :5678
2026/11/23 13:17:40 api-service 10.233.127.207:36722 "GET / HTTP/1.1" 200 29 "curl/8.5.0" 17.465µs

# api-beta
2026/11/23 13:14:44 [INFO] server is listening on :8080
```

| Pod | Port d'écoute | Requêtes reçues |
|---|---|---|
| `api-alpha-...` | 5678 | Oui |
| `api-beta-...` | 8080 | Aucune |

Beta écoute sur le mauvais port : le Service envoie sur 5678, donc les pods Beta refusent la connexion.

### Cause racine

Dans `api-beta`, `-listen=:8080` est incohérent avec le `targetPort: 5678` du Service.

## Étape 3 : résolution

### Modification

Je corrige Beta pour l'aligner sur Alpha et sur le Service (plutôt que de modifier le Service, qui n'a qu'un seul `targetPort`) :

```diff
-            - "-listen=:8080"
+            - "-listen=:5678"
```

### Commandes et résultats

```bash
kubectl apply -f debug-lab-service.yaml
kubectl get pods
```

je relance les requêtes :

```
~ $ curl http://api-service
Reponse du deploiement Beta
```

Après la correction, Beta répond (ce qui était impossible avant) et toutes les requêtes aboutissent, avec un mélange de réponses Alpha et Beta.


## Difficultés rencontrées

## Difficultés rencontrées

Je ne savais pas trop par où commencer. Le lab dit seulement qu'une requête sur deux échoue, et tous les Pods étaient en `Running`, donc rien ne signalait d'erreur.
