# TP 1 : Conteneurisation

## Exercice 1 : construire et exécuter une API Python

### Dockerfile

```dockerfile
FROM python:3.12-slim

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY src/ .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
```

### Layers importantes

### Layers importantes

| Instruction | Rôle |
|---|---|
| `FROM python:3.12-slim` | Permet de choisir une image de base |
| `COPY requirements.txt .` | Copie le fichier des dépendances du répertoire courant dans l'image lors de la construction. |
| `RUN pip install -r requirements.txt` | Lance une commande de configuration : l'installation des dépendances. |
| `COPY src/ .` | Copie le code de l'application dans l'image. |
| `EXPOSE 8000` | Permet de configurer le port exposé par le conteneur. (Documentation) |
| `CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]` | Définit la commande par défaut du conteneur, puisqu'aucun `ENTRYPOINT` n'est spécifié. |
### Commandes et résultats

**Build**

```bash
docker build -t python-api .
```

**Exécution** (le port 8000 du conteneur est publié sur le port 8000 de la machine)

```bash
docker run -d --name python-api -p 8000:8000 python-api
```

**Vérification que le conteneur tourne**

```
$ docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                    NAMES
29b570d51005   python-api   "uvicorn main:app --…"   35 seconds ago   Up 34 seconds   0.0.0.0:8000->8000/tcp   python-api
```

**Appel de l'API**

```
$ curl http://localhost:8000/hello
"world"
```

**Logs**

```
$ docker logs python-api
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     172.17.0.1:55276 - "GET /hello HTTP/1.1" 200 OK
```

### Image sur Docker Hub

L'image est disponible ici : https://hub.docker.com/r/lucasrt6/python-api

```bash
docker tag python-api lucasrt6/python-api:1.0
docker push lucasrt6/python-api:1.0
```

### Difficultés rencontrées

Je ne connaissais pas bien les instructions Docker, notamment les paramètres de `CMD` (comment lancer uvicorn avec `main:app` et `--host 0.0.0.0`) pour écrire le Dockerfile. J'ai eu la même difficulté pour les commandes de création et d'utilisation de l'image (`docker build`, `docker run -p`, `docker tag`, `docker push`). Pour y arriver, je me suis beaucoup appuyé sur la correction, et un peu sur une IA pour comprendre les instructions et les erreurs rencontrées.


## Exercice 2 : API Java, optimisation et multi-stage build

### Dockerfile

```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package

FROM eclipse-temurin:21-jre
COPY --from=build /app/target/java-api-1.0.0.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### Layers importantes

| Instruction | Rôle |
|---|---|
| `FROM maven:3.9-eclipse-temurin-21 AS build` | Permet de choisir une image de base : ici une image avec Maven et un JDK 21 |
| `COPY pom.xml .` puis `RUN mvn dependency:go-offline` | Copie le `pom.xml` dans l'image, puis lance une commande de configuration : le téléchargement des dépendances. |
| `COPY src ./src` puis `RUN mvn package` | Copie le code source, puis lance la compilation qui produit le jar. |
| `FROM eclipse-temurin:21-jre` | Second stage : une image de base avec un JRE seulement |
| `COPY --from=build ... app.jar` | Copie uniquement le jar depuis le stage `build` |
| `EXPOSE 8080` | Documente le port d'écoute de l'application. |
| `CMD ["java", "-jar", "app.jar"]` | Définit la commande par défaut du conteneur, puisqu'aucun `ENTRYPOINT` n'est spécifié. |

### Commandes et résultats

```bash
docker build -t java-api .
docker run -d --name java-api -p 8080:8080 java-api
```

**Logs** (extrait)

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.2.0)

INFO 1 --- [main] com.example.Application : Starting Application v1.0.0 using Java 21.0.13.1
INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer : Tomcat initialized with port 8080 (http)
INFO 1 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer : Tomcat started on port 8080 (http) with context path ''
INFO 1 --- [main] com.example.Application : Started Application in 2.614 seconds
INFO 1 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet : Completed initialization in 2 ms
```

**Appel de l'API**

```
$ curl http://localhost:8080/hello
world
```

### Comparaison avec un build en un seul stage

Pour comparer, j'ai construit une seconde image avec un `Dockerfile.single` qui compile et exécute dans la même image :

```dockerfile
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package
EXPOSE 8080
CMD ["java", "-jar", "target/java-api-1.0.0.jar"]
```

Maven/JDK :

```bash
docker build -f Dockerfile.single -t java-api:single .
docker images java-api
```

```
IMAGE             ID             DISK USAGE   CONTENT SIZE
java-api:latest   aaffd5f31043        517MB          140MB
java-api:single   b887878637aa        996MB          326MB
```

| Image | Disk usage | Content size |
|---|---|---|
| Multi-stage (JRE) | 517 Mo | 140 Mo |
| Mono-stage (Maven + JDK) | 996 Mo | 326 Mo |

L'image multi-stage est environ deux fois plus petite. L'image mono-stage embarque tout ce qui a servi à la construction : Maven, le JDK complet, les sources, le dossier `target` et le cache des dépendances. À l'exécution, seul le jar est utile. Le multi-stage garde uniquement le JRE et le jar, ce qui réduit la taille, le temps de téléchargement et la surface d'attaque (moins de logiciels installés dans l'image de production).

### Image sur Docker Hub

https://hub.docker.com/r/lucasrt6/java-api

```bash
docker tag java-api lucasrt6/java-api:1.0
docker push lucasrt6/java-api:1.0
```

### Difficultés rencontrées

Comprendre le multi-stage build.

Comprendre le rôle du `Dockerfile.single`.
