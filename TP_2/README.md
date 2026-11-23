# TP 2 : Premiers pas sur Kubernetes


## Manifest

```yaml
# Version de l'API Kubernetes pour cet objet
apiVersion: apps/v1
# Un Deployment gère un ensemble de Pods identiques
kind: Deployment
metadata:
  # Nom du Deployment
  name: hello-api
spec:
  # Nombre de Pods à maintenir en permanence
  replicas: 3
  # Le Deployment retrouve ses Pods grâce à ce label
  selector:
    matchLabels:
      app: hello-api
  # Modèle utilisé pour créer chaque Pod
  template:
    metadata:
      # Label des Pods : doit correspondre au selector
      labels:
        app: hello-api
    spec:
      containers:
        # Nom du conteneur
        - name: hello-api
          # Image publiée sur Docker Hub au TP 1
          image: lucasrt6/python-api:1.0
          ports:
            # Port d'écoute de l'API (informatif)
            - containerPort: 8000
```

Un Deployment est utilisé plutôt qu'un Pod seul : il maintient les 3 replicas et recrée automatiquement un Pod supprimé.

### Rôle des éléments principaux

| Élément | Rôle |
|---|---|
| `kind: Deployment` | Choisi plutôt qu'un Pod seul : il maintient le nombre de replicas voulu et recrée les Pods disparus. |
| `replicas: 3` | Nombre de Pods identiques demandé par l'énoncé. |
| `selector.matchLabels` | Permet au Deployment de retrouver les Pods qui lui appartiennent. |
| `template.metadata.labels` | Labels posés sur chaque Pod créé. Ils doivent correspondre au `selector`, sinon le Deployment est rejeté. |
| `image: lucasrt6/python-api:1.0` | Image du TP 1, avec un tag précis (et non `latest`) pour savoir exactement quelle version tourne. |
| `containerPort: 8000` | Port d'écoute d'uvicorn. Il est purement informatif : il n'expose rien à lui seul. |

## Déploiement

```bash
kubectl apply -f deployment.yaml
```

```
deployment.apps/hello-api created
```

```
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
hello-api-76fcb58f46-6ths8   1/1     Running   0          22s
hello-api-76fcb58f46-mdvqn   1/1     Running   0          22s
hello-api-76fcb58f46-stb84   1/1     Running   0          23s
vscode-python-74536-0        1/1     Running   0          3m39s

$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
hello-api   3/3     3            3            67s
```

Le Pod `vscode-python-74536-0` est l'environnement VSCode de SSP Cloud dans lequel je travaille, il n'a rien à voir avec le déploiement.

## Tester l'application sans Service

L'application n'est pas exposée sur le réseau. Pour la tester, j'ai utilisé `kubectl port-forward`, qui redirige un port de ma machine vers un Pod du Deployment :

```bash
kubectl port-forward deployment/hello-api 8000:8000
```

Dans un second terminal :

```
$ curl http://localhost:8000/hello
"world"
```

### Observation : pas de répartition de charge

En consultant les logs des trois Pods, seul `hello-api-76fcb58f46-stb84` a reçu des requêtes :

```
INFO:     127.0.0.1:40356 - "GET /hello HTTP/1.1" 200 OK
```

Les deux autres n'affichent que leur démarrage. Un `port-forward` sur un Deployment ne cible qu'un seul Pod : il ne répartit pas la charge entre les 3 replicas.

## Auto-réparation

J'ai supprimé un des Pods pour vérifier que le Deployment maintient bien 3 replicas :

```
$ kubectl delete pod hello-api-76fcb58f46-6ths8
pod "hello-api-76fcb58f46-6ths8" deleted

$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
hello-api-76fcb58f46-d7l76   1/1     Running   0          8s
hello-api-76fcb58f46-mdvqn   1/1     Running   0          3m49s
hello-api-76fcb58f46-stb84   1/1     Running   0          3m50s
vscode-python-74536-0        1/1     Running   0          7m6s
```

Un nouveau Pod (`d7l76`) a été créé en quelques secondes pour remplacer celui supprimé, tandis que les deux autres n'ont pas bougé. Le Deployment compare en permanence l'état réel à l'état souhaité (`replicas: 3`) et corrige l'écart.

## Difficultés rencontrées

La principale difficulté a été de comprendre la structure d'un manifest Kubernetes et le fichier `deployment.yaml` : le rôle de `selector` et des `labels`, ou encore la différence entre le Deployment et le `template` des Pods. Je n'ai pas tout compris du premier coup. Pour écrire le manifest, j'ai utilisé une IA, ce qui m'a aidé à comprendre sa syntaxe et à commenter chaque élément.
