------------------------------------------------------------------------------------------------------
ATELIER PRA/PCA
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier met en œuvre un **mini-PRA** sur **Kubernetes** en déployant une **application Flask** avec une **base SQLite** stockée sur un **volume persistant (PVC pra-data)** et des **sauvegardes automatiques réalisées chaque minute vers un second volume (PVC pra-backup)** via un **CronJob**. L’**image applicative est construite avec Packer** et le **déploiement orchestré avec Ansible**, tandis que Kubernetes assure la gestion des pods et de la disponibilité applicative. Nous observerons la différence entre **disponibilité** (recréation automatique des pods sans perte de données) et **reprise après sinistre** (perte volontaire du volume de données puis restauration depuis les backups), nous mesurerons concrètement les RTO et RPO, et comprendrons les limites d’un PRA local non répliqué. Cet atelier illustre de manière pratique les principes de continuité et de reprise d’activité, ainsi que le rôle respectif des conteneurs, du stockage persistant et des mécanismes de sauvegarde.
  
**Architecture cible :** Ci-dessous, voici l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)  
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besoin, voici une vidéo d'accompagnement pour vous aider à "Forker" un Repository Github : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet **[CODE]** de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du votre environnement de travail
---------------------------------------------------
Objectif : Créer votre environnement de travail  
Difficulté : Simple (~10 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers, installer les logiciels Packer et Ansible. Depuis le terminal de votre Codespace copier/coller les codes ci-dessous étape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create pra \
  --servers 1 \
  --agents 2
```
**vérification de la création de votre cluster Kubernetes**  
```
kubectl get nodes
```
**Installation du logiciel Packer (création d'images Docker)**  
```
PACKER_VERSION=1.11.2
curl -fsSL -o /tmp/packer.zip \
  "https://releases.hashicorp.com/packer/${PACKER_VERSION}/packer_${PACKER_VERSION}_linux_amd64.zip"
sudo unzip -o /tmp/packer.zip -d /usr/local/bin
rm -f /tmp/packer.zip
```
**Installation du logiciel Ansible**  
```
python3 -m pip install --user ansible kubernetes PyYAML jinja2
export PATH="$HOME/.local/bin:$PATH"
ansible-galaxy collection install kubernetes.core
```
  
---------------------------------------------------
Séquence 3 : Déploiement de l'infrastructure
---------------------------------------------------
Objectif : Déployer l'infrastructure sur le cluster Kubernetes
Difficulté : Facile (~15 minutes)
---------------------------------------------------  
Nous allons à présent déployer notre infrastructure sur Kubernetes. C'est à dire, créér l'image Docker de notre application Flask avec Packer, déposer l'image dans le cluster Kubernetes et enfin déployer l'infratructure avec Ansible (Création du pod, création des PVC et les scripts des sauvegardes aututomatiques).  

**Création de l'image Docker avec Packer**  
```
packer init .
packer build -var "image_tag=1.0" .
docker images | head
```
  
**Import de l'image Docker dans le cluster Kubernetes**  
```
k3d image import pra/flask-sqlite:1.0 -c pra
```
  
**Déploiment de l'infrastructure dans Kubernetes**  
```
ansible-playbook ansible/playbook.yml
```
  
**Forward du port 8080 qui est le port d'exposition de votre application Flask**  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
  
---------------------------------------------------  
**Réccupération de l'URL de votre application Flask**. Votre application Flask est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace (à coté de Terminal) et rendez public votre port 8080 (Visibilité du port). Ouvrez l'URL dans votre navigateur et c'est terminé.  

**Les routes** à votre disposition sont les suivantes :  
1. https://...**/** affichera dans votre navigateur "Bonjour tout le monde !".
2. https://...**/health** pour voir l'état de santé de votre application.
3. https://...**/add?message=test** pour ajouter un message dans votre base de données SQLite.
4. https://...**/count** pour afficher le nombre de messages stockés dans votre base de données SQLite.
5. https://...**/consultation** pour afficher les messages stockés dans votre base de données.
  
---------------------------------------------------  
### Processus de sauvegarde de la BDD SQLite

Grâce à une tâche CRON déployée par Ansible sur le cluster Kubernetes (un CronJob), toutes les minutes une sauvegarde de la BDD SQLite est faite depuis le PVC pra-data vers le PCV pra-backup dans Kubernetes.  

Pour visualiser les sauvegardes périodiques déposées dans le PVC pra-backup, coller les commandes suivantes dans votre terminal Codespace :  

```
kubectl -n pra run debug-backup \
  --rm -it \
  --image=alpine \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh"],
      "stdin": true,
      "tty": true,
      "volumeMounts": [{
        "name": "backup",
        "mountPath": "/backup"
      }]
    }],
    "volumes": [{
      "name": "backup",
      "persistentVolumeClaim": {
        "claimName": "pra-backup"
      }
    }]
  }
}'
```
```
ls -lh /backup
```
**Pour sortir du cluster et revenir dans le terminal**
```
exit
```

---------------------------------------------------
Séquence 4 : 💥 Scénarios de crash possibles  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
### 🎬 **Scénario 1 : PCA — Crash du pod**  
Nous allons dans ce scénario **détruire notre Pod Kubernetes**. Ceci simulera par exemple la supression d'un pod accidentellement, ou un pod qui crash, ou un pod redémarré, etc..

**Destruction du pod :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario1.png)  

Nous perdons donc ici notre application mais pas notre base de données puisque celle-ci est déposée dans le PVC pra-data hors du pod.  

Copier/coller le code suivant dans votre terminal Codespace pour détruire votre pod :
```
kubectl -n pra get pods
```
Notez le nom de votre pod qui est différent pour tout le monde.  
Supprimez votre pod (pensez à remplacer <nom-du-pod-flask> par le nom de votre pod).  
Exemple : kubectl -n pra delete pod flask-7c4fd76955-abcde  
```
kubectl -n pra delete pod <nom-du-pod-flask>
```
**Vérification de la suppression de votre pod**
```
kubectl -n pra get pods
```
👉 **Le pod a été reconstruit sous un autre identifiant**.  
Forward du port 8080 du nouveau service  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
Observez le résultat en ligne  
https://...**/consultation** -> Vous n'avez perdu aucun message.
  
👉 Kubernetes gère tout seul : Aucun impact sur les données ou sur votre service (PVC conserve la DB et le pod est reconstruit automatiquement) -> **C'est du PCA**. Tout est automatique et il n'y a aucune rupture de service.
  
---------------------------------------------------
### 🎬 **Scénario 2 : PRA - Perte du PVC pra-data** 
Nous allons dans ce scénario **détruire notre PVC pra-data**. C'est à dire nous allons suprimer la base de données en production. Ceci simulera par exemple la corruption de la BDD SQLite, le disque du node perdu, une erreur humaine, etc. 💥 Impact : IL s'agit ici d'un impact important puisque **la BDD est perdue**.  

**Destruction du PVC pra-data :** Ci-dessous, la cible de notre scénario   
  
![Screenshot Actions](scenario2.png)  

🔥 **PHASE 1 — Simuler le sinistre (perte de la BDD de production)**  
Copier/coller le code suivant dans votre terminal Codespace pour détruire votre base de données :
```
kubectl -n pra scale deployment flask --replicas=0
```
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```
```
kubectl -n pra delete job --all
```
```
kubectl -n pra delete pvc pra-data
```
👉 Vous pouvez vérifier votre application en ligne, la base de données est détruite et la service n'est plus accéssible.  

✅ **PHASE 2 — Procédure de restauration**  
Recréer l’infrastructure avec un PVC pra-data vide.  
```
kubectl apply -f k8s/
```
Vérification de votre application en ligne.  
Forward du port 8080 du service pour tester l'application en ligne.  
```
kubectl -n pra port-forward svc/flask 8080:80 >/tmp/web.log 2>&1 &
```
https://...**/count** -> =0.  
https://...**/consultation** Vous avez perdu tous vos messages.  

Retaurez votre BDD depuis le PVC Backup.  
```
kubectl apply -f pra/50-job-restore.yaml
```
👉 Vous pouvez vérifier votre application en ligne, **votre base de données a été restaureé** et tous vos messages sont bien présents.  

Relance des CRON de sauvgardes.  
```
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```
👉 Nous n'avons pas perdu de données mais Kubernetes ne gère pas la restauration tout seul. Nous avons du protéger nos données via des sauvegardes régulières (du PVC pra-data vers le PVC pra-backup). -> **C'est du PRA**. Il s'agit d'une stratégie de sauvegarde avec une procédure de restauration.  

---------------------------------------------------
Séquence 5 : Exercices  
Difficulté : Moyenne (~45 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour répondre aux questions des exercices.  
Faites preuve de pédagogie et soyez clair dans vos explications et procedures de travail.  

**Exercice 1 :**  
Quels sont les composants dont la perte entraîne une perte de données ?  
  
Les composants dont la perte entraine une perte de donnees sont : le **PVC pra-data** (qui contient la base SQLite de production) et le **PVC pra-backup** (qui contient les sauvegardes). Si le PVC pra-data est perdu sans que pra-backup existe, les donnees sont definitivement perdues. Le pod lui-meme est sans etat (stateless) et peut etre recree sans perte de donnees.

**Exercice 2 :**  
Expliquez nous pourquoi nous n'avons pas perdu les données lors de la supression du PVC pra-data  
  
Lors de la suppression du pod (scenario PCA), nous n'avons pas perdu les donnees car celles-ci sont stockees sur un **PersistentVolumeClaim (PVC)** et non dans le pod. Le PVC `pra-data` survit a la destruction du pod. Quand Kubernetes recree automatiquement le pod via le Deployment, le nouveau pod remonte le meme PVC et retrouve donc toutes les donnees intactes.

**Exercice 3 :**  
Quels sont les RTO et RPO de cette solution ?  
  
**RPO (Recovery Point Objective)** = 1 minute maximum, car le CronJob de sauvegarde s'execute toutes les minutes. On peut donc perdre au maximum 1 minute de donnees. **RTO (Recovery Time Objective)** = quelques minutes, correspondant au temps de detection de la panne, de suppression de l'ancien PVC, de recreation du PVC, d'execution du Job de restauration et de redemarrage du pod.

**Exercice 4 :**  
Pourquoi cette solution (cet atelier) ne peux pas être utilisé dans un vrai environnement de production ? Que manque-t-il ?   
  
Cette solution ne peut pas etre utilisee en production car : (1) les backups sont stockes sur le meme noeud/cluster que les donnees de production, donc une panne du noeud entraine la perte des deux. Il faudrait un stockage distant (S3, NFS). (2) SQLite ne supporte pas les acces concurrents, il faudrait une vraie base de donnees (PostgreSQL, MySQL). (3) Il n'y a pas de monitoring ni d'alerting automatise. (4) Le RTO est manuel et depend d'une intervention humaine. (5) Il n'y a pas de replication multi-site ni de haute disponibilite.
  
**Exercice 5 :**  
Proposez une archtecture plus robuste.   
  
Pour une architecture plus robuste en production, voici les ameliorations proposees :

**1. Replication multi-site (Disaster Recovery)**
- Deployer un cluster Kubernetes secondaire sur un site geographiquement distant (ou une seconde region cloud).
- Utiliser un outil comme **Velero** pour repliquer les sauvegardes vers un stockage objet distant (S3, Azure Blob, GCS).
- Mettre en place un DNS failover (Route53, Cloudflare) pour basculer automatiquement vers le site secondaire.

**2. Base de donnees adaptee**
- Remplacer SQLite par **PostgreSQL** avec replication synchrone (primary/standby).
- Utiliser un operateur Kubernetes comme **CloudNativePG** ou **Zalando Postgres Operator** pour gerer le failover automatique.

**3. Haute disponibilite applicative**
- Deployer l'application avec **minimum 2 replicas** repartis sur des nodes differents (`podAntiAffinity`).
- Utiliser un **Ingress Controller** avec health checks pour router le trafic uniquement vers les pods sains.

**4. Stockage distribue**
- Remplacer le stockage local par un systeme distribue comme **Longhorn**, **Rook-Ceph** ou un stockage cloud manage.
- Activer la replication des volumes sur plusieurs nodes.

**5. Monitoring et alerting**
- Deployer **Prometheus + Grafana** pour surveiller l'etat du cluster, des pods et des backups.
- Configurer des alertes (PagerDuty, Slack) en cas d'echec de backup ou de pod en erreur.

**6. Automatisation du PRA**
- Ecrire des runbooks automatises (scripts ou pipelines CI/CD) pour la restauration.
- Tester regulierement le PRA avec des exercices de simulation (Chaos Engineering avec **LitmusChaos** ou **Chaos Monkey**).

**7. Securite des sauvegardes**
- Chiffrer les backups au repos et en transit.
- Verifier l'integrite des sauvegardes avec des checksums.
- Appliquer une politique de retention (ex: 7 jours de backups quotidiens, 4 semaines de backups hebdomadaires).

---------------------------------------------------
Séquence 6 : Ateliers  
Difficulté : Moyenne (~2 heures)
---------------------------------------------------
### **Atelier 1 : Ajoutez une fonctionnalité à votre application**  
**Ajouter une route GET /status** dans votre application qui affiche en JSON :
* count : nombre d’événements en base
* last_backup_file : nom du dernier backup présent dans /backup
* backup_age_seconds : âge du dernier backup

Voici le resultat de la route `/status` :

```json
{"backup_age_seconds":null,"count":3,"last_backup_file":null}
```

La route retourne bien en JSON :
- `count` : le nombre d'evenements en base (ici 3)
- `last_backup_file` : le nom du dernier fichier de backup present dans /backup
- `backup_age_seconds` : l'age en secondes du dernier backup

Le code de la route est dans `app/app.py` et utilise `glob` pour lister les fichiers de backup dans `/backup/`, `os.path.getmtime()` pour calculer l'age du backup, et une requete SQL `SELECT COUNT(*)` pour le nombre d'evenements.

---------------------------------------------------
### **Atelier 2 : Choisir notre point de restauration**  
Aujourd’hui nous restaurobs “le dernier backup”. Nous souhaitons **ajouter la capacité de choisir un point de restauration**.

### Runbook : Restauration a un point precis

**Objectif** : Permettre a l'operateur de choisir un backup specifique plutot que de restaurer systematiquement le dernier backup.

**Pre-requis** : Le fichier `pra/51-job-restore-select.yaml` a ete cree pour cette procedure.

**Etape 1 : Lister les backups disponibles**

Lancer un pod temporaire pour voir les backups :
```bash
kubectl -n pra run debug-backup --rm -it --image=alpine --overrides='
{
  "spec": {
    "containers": [{
      "name": "debug",
      "image": "alpine",
      "command": ["sh", "-c", "ls -lh /backup/app-*.db"],
      "volumeMounts": [{"name": "backup", "mountPath": "/backup"}]
    }],
    "volumes": [{"name": "backup", "persistentVolumeClaim": {"claimName": "pra-backup"}}]
  }
}'
```

Cela affichera une liste de fichiers comme :
```
-rw-r--r--  1 root  root  20K  app-1740000000.db
-rw-r--r--  1 root  root  20K  app-1740000060.db
-rw-r--r--  1 root  root  20K  app-1740000120.db
```

Le timestamp Unix dans le nom du fichier indique le moment de la sauvegarde. On peut le convertir avec : `date -d @1740000000`

**Etape 2 : Arreter l'application et les sauvegardes**

```bash
kubectl -n pra scale deployment flask --replicas=0
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":true}}'
```

**Etape 3 : Choisir et restaurer le backup**

Editer le fichier `pra/51-job-restore-select.yaml` et remplacer `REPLACE_ME` par le nom du fichier choisi (ex: `app-1740000060.db`) :

```bash
sed -i 's/REPLACE_ME/app-1740000060.db/' pra/51-job-restore-select.yaml
kubectl delete job sqlite-restore-select -n pra --ignore-not-found
kubectl apply -f pra/51-job-restore-select.yaml
```

**Etape 4 : Relancer l'application et les sauvegardes**

```bash
kubectl -n pra scale deployment flask --replicas=1
kubectl -n pra patch cronjob sqlite-backup -p '{"spec":{"suspend":false}}'
```

**Etape 5 : Verifier la restauration**

```bash
kubectl -n pra port-forward svc/flask 8080:80 &
curl -s localhost:8080/consultation
curl -s localhost:8080/count
```

Verifier que les donnees correspondent au point de restauration choisi.

**Resume de la procedure** :
1. Lister les backups et identifier le point de restauration souhaite
2. Stopper l'application et le CronJob
3. Lancer le Job de restauration avec le fichier choisi
4. Relancer l'application et le CronJob
5. Verifier les donnees restaurees  
  
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier PRA PCA, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Série d'exerices (5 points)
- Atelier N°1 - Ajout d'un fonctionnalité (4 points)
- Atelier N°2 - Choisir son point de restauration (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (3 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 

