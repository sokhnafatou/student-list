# 🐳 Student List — Conteneurisation avec Docker

> **Projet POZOS** — Preuve de concept (POC) démontrant l'efficacité de Docker pour moderniser l'infrastructure de déploiement d'une application web.

---

## 📋 Table des matières

- [Contexte](#-contexte)
- [Architecture](#-architecture)
- [Structure du dépôt](#-structure-du-dépôt)
- [Étape 1 — Build de l'image API](#-étape-1--build-de-limage-api)
- [Étape 2 — Déploiement avec Docker Compose](#-étape-2--déploiement-avec-docker-compose)
- [Étape 3 — Registre Docker privé](#-étape-3--registre-docker-privé)

---

## 🏢 Contexte

**POZOS** est une entreprise informatique française qui développe des logiciels pour les lycées. Son département innovation souhaite moderniser son infrastructure en adoptant Docker pour assurer :

- La **scalabilité** de ses applications
- La **facilité de déploiement** via l'automatisation
- La **reproductibilité** des environnements

L'application `student_list` est une application simple composée de deux modules :

| Module | Technologie | Rôle |
|--------|------------|------|
| **API REST** | Python / Flask | Retourne la liste des étudiants depuis un fichier JSON (authentification requise) |
| **Application Web** | PHP / Apache | Interface utilisateur pour consulter la liste des étudiants |

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────┐
│              Réseau Docker : pozos_network        │
│                                                   │
│  ┌─────────────────┐      ┌──────────────────┐   │
│  │  webapp          │      │   pozos_api       │   │
│  │  php:apache      │─────▶│   student-api     │   │
│  │  Port: 80        │      │   Port: 5000      │   │
│  └─────────────────┘      └────────┬─────────┘   │
│                                     │             │
│                            ┌────────▼─────────┐   │
│                            │  Volume           │   │
│                            │  student_age.json │   │
│                            └──────────────────┘   │
└─────────────────────────────────────────────────┘
        │                          │
      :80                        :5000
   Navigateur                   curl / API
```

---

## 📁 Structure du dépôt

```
student-list/
├── simple_api/
│   ├── Dockerfile            # Image de l'API Flask
│   ├── student_age.py        # Code source de l'API
│   ├── student_age.json      # Données des étudiants
│   └── requirements.txt      # Dépendances Python
├── website/
│   └── index.php             # Interface web PHP
├── registry/
│   └── auth/                 # Authentification du registre privé
├── docker-compose.yml        # Déploiement API + Web
└── docker-compose-registry.yml  # Déploiement registre privé
```

---

## 🔨 Étape 1 — Build de l'image API

### Le Dockerfile

```dockerfile
FROM python:3.13-slim
LABEL maintainer="EAZYTraining"
WORKDIR /app
COPY student_age.py .
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
        python3-dev \
        libsasl2-dev \
        libldap2-dev \
        libssl-dev \
        gcc \
        build-essential && \
    rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip3 install -r requirements.txt
VOLUME /data
EXPOSE 5000
CMD [ "python3", "./student_age.py" ]
```

**Explication des instructions :**

| Instruction | Rôle |
|-------------|------|
| `FROM python:3.13-slim` | Image de base légère Python 3.13 |
| `LABEL maintainer` | Métadonnée d'identification du mainteneur |
| `WORKDIR /app` | Définit le répertoire de travail dans le conteneur |
| `COPY student_age.py .` | Copie le code source de l'API |
| `RUN apt-get install ...` | Installe les dépendances système nécessaires |
| `RUN pip3 install` | Installe les bibliothèques Python |
| `VOLUME /data` | Déclare le point de montage pour les données |
| `EXPOSE 5000` | Expose le port de l'API |
| `CMD` | Commande de démarrage du conteneur |

### Construction de l'image

```bash
docker build -t student-api ./simple_api/
```

> 📸 **Capture 1 — Build de l'image**
> 
> ![Build de l'image Docker](screenshots/01-docker-build.png)

### Test de l'API

Lancement du conteneur :

```bash
docker run -d \
  --name api \
  -p 5000:5000 \
  -v "D:/path/to/simple_api/student_age.json:/data/student_age.json" \
  student-api
```

Vérification que l'API répond :

```bash
curl -u toto:python http://localhost:5000/pozos/api/v1.0/get_student_ages
```

Réponse attendue :

```json
{
  "student_ages": {
    "alice": "12",
    "bob": "13"
  }
}
```

> 📸 **Capture 2 — Test curl de l'API**
> 
> ![Test curl de l'API](screenshots/02-curl-api.png)

---

## 🐙 Étape 2 — Déploiement avec Docker Compose

### Le docker-compose.yml

```yaml
version: '3.3'
services:
  webapp_student_list:
    image: php:apache
    container_name: webapp_student_list
    depends_on:
      - pozos_api
    ports:
      - "80:80"
    volumes:
      - ./website/:/var/www/html
    environment:
      - USERNAME=toto
      - PASSWORD=python
    networks:
      - pozos_network

  pozos_api:
    image: student-api
    container_name: pozos_api
    volumes:
      - ./simple_api/student_age.json:/data/student_age.json
    ports:
      - "5000:5000"
    networks:
      - pozos_network

networks:
  pozos_network:
    name: pozos_network
    driver: bridge
```

**Points clés de la configuration :**

- `depends_on` : garantit que l'API démarre avant l'application web
- `networks` : les deux services communiquent via un réseau Docker dédié (`pozos_network`), isolé du reste
- Les variables d'environnement `USERNAME` et `PASSWORD` permettent à l'application web de s'authentifier auprès de l'API
- Le volume monte directement le fichier JSON dans le conteneur API

### Déploiement

```bash
docker compose up -d
```

> 📸 **Capture 3 — Docker Compose up**
> 
> ![Docker Compose démarrage](screenshots/03-docker-compose-up.png)

### Test de l'interface web

Accéder à **http://localhost** et cliquer sur **"List Student"**.

> 📸 **Capture 4 — Interface web avec la liste des étudiants**
> 
> ![Interface web POZOS](screenshots/04-web-interface.png)

---

## 📦 Étape 3 — Registre Docker privé

Un registre Docker privé est déployé avec une interface web pour gérer les images.

### Le docker-compose-registry.yml

```yaml
version: '3.3'
services:
  pozos-registry:
    image: registry:2.8.1
    container_name: pozos-registry
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - /opt/docker/registry:/var/lib/registry
      - ./registry/auth:/auth
    environment:
      - REGISTRY_STORAGE_DELETE_ENABLED=true
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
    networks:
      - pozos_reg_network

  frontend-reg:
    image: joxit/docker-registry-ui:2
    container_name: frontend-reg
    depends_on:
      - pozos-registry
    ports:
      - "8080:80"
    environment:
      - NGINX_PROXY_PASS_URL=http://pozos-registry:5000
      - DELETE_IMAGES=true
      - REGISTRY_TITLE=Pozos
      - SINGLE_REGISTRY=true
    networks:
      - pozos_reg_network

networks:
  pozos_reg_network:
    driver: bridge
```

### Déploiement du registre

```bash
docker compose -f docker-compose-registry.yml up -d
```

### Push de l'image vers le registre privé

```bash
# Connexion au registre
docker login localhost:5000 -u pozos -p change-me

# Tag de l'image
docker tag student-api localhost:5000/pozos/student-api:1.0

# Push vers le registre
docker push localhost:5000/pozos/student-api:1.0
```

> 📸 **Capture 5 — Interface du registre privé (vide)**
> 
> ![Registre privé vide](screenshots/05-registry-empty.png)

> 📸 **Capture 6 — Image poussée dans le registre**
> 
> ![Image dans le registre](screenshots/06-registry-with-image.png)

---

## ✅ Résumé

| Étape | Description | Statut |
|-------|-------------|--------|
| Build Dockerfile | Image API Flask construite | ✅ |
| Test API | curl retourne la liste des étudiants | ✅ |
| Docker Compose | API + Web déployés ensemble | ✅ |
| Interface web | Liste des étudiants visible | ✅ |
| Registre privé | Déployé avec interface graphique | ✅ |
| Push image | student-api disponible dans le registre | ✅ |

---

*Réalisé dans le cadre du cours DevOps — Docker — EazyTraining*