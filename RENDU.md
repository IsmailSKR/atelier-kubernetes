# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?

   > k3d-lab-server-0 (control-plane) : c'est le nœud maître (master) qui gère l'orchestration du cluster (API Server, planification des pods, maintien de l'état désiré). k3d-lab-agent-0 et k3d-lab-agent-1 : ce sont les deux nœuds workers chargés d'exécuter les conteneurs applicatifs.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   >docker ps montre 4 conteneurs actifs : les 3 nœuds du cluster (k3d-lab-server-0, k3d-lab-agent-0, k3d-lab-agent-1) ainsi qu'un quatrième conteneur k3d-lab-serverlb qui sert de load balancer (proxy) en entrée du cluster. Concrètement, dans k3d, un « nœud » Kubernetes est simplement un conteneur Docker exécutant une instance légère de Kubernetes (k3s).

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   >La commande kubectl create deployment a créé trois objets liés entre eux : Un Deployment (deployment.apps/web) qui définit l'état désiré de l'application (1 réplica de l'image nginx). Un ReplicaSet (replicaset.apps/web-68d995574f) créé par le Deployment pour maintenir en permanence le bon nombre de pods actifs. Un Pod (pod/web-...) qui exécute concrètement le conteneur nginx. (Le service/web a quant à lui été créé juste après par kubectl expose).
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   >Dès que le pod a été supprimé (Terminating), un nouveau pod avec un identifiant différent a été créé et démarré automatiquement (Running). Cela s'explique par la boucle de réconciliation de Kubernetes : le ReplicaSet surveille en continu l'état réel du cluster et, constatant qu'il y avait 0 pod actif au lieu du 1 réplica demandé dans l'état désiré, il en a immédiatement relancé un nouveau.

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   >Non, le message n'a pas survécu : après la recréation du pod, c'est la page par défaut « Welcome to nginx! » qui réapparaît. Cela s'explique par le fait que les conteneurs sont éphémères et que leur système de fichiers n'est pas persistant : lorsqu'un pod est supprimé, le ReplicaSet en crée un tout nouveau à partir de l'image nginx d'origine, effaçant toute modification effectuée directement à l'intérieur.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   >Même s'il est lancé en ciblant le Service (svc/web), un kubectl port-forward établit en réalité une connexion réseau directe vers un pod spécifique choisi derrière ce Service. Dès que ce pod est supprimé, le tunnel perd sa cible et s'interrompt (lost connection to pod).

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   >kubectl create est une commande impérative : elle crée une ressource neuve une seule fois et échoue avec une erreur (AlreadyExists) si l'objet existe déjà. kubectl apply est une commande déclarative : elle lit l'état désiré dans un fichier YAML, crée la ressource si elle n'existe pas encore, ou met à jour uniquement les différences si la ressource existe déjà (ce qui permet de versionner toute l'infrastructure dans Git).
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   >Le selector d'un Service sert à identifier et cibler dynamiquement les pods (via leurs labels) vers lesquels il doit rediriger le trafic réseau. S'il ne correspond à aucun pod, le Service existe toujours mais n'a aucun Endpoint associé (kubectl get endpoints affiche <none>), et toute requête envoyée vers ce Service échoue (connexion refusée ou timeout).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.

NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
site-75775484b-g7jj5   1/1     Running   0          10s     10.42.2.6   k3d-lab-agent-1    <none>           <none>
site-75775484b-p8ksk   1/1     Running   0          2m20s   10.42.0.9   k3d-lab-server-0   <none>           <none>
site-75775484b-pkwnd   1/1     Running   0          10s     10.42.1.5   k3d-lab-agent-0    <none>           <none>

10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod prenait le statut ErrImagePull puis ImagePullBackOff. C'est la commande kubectl describe pod -l app=site (section Events) qui a donné la cause exacte : le téléchargement de l'image nginx:9.99 échoue car ce tag n'existe pas sur Docker Hub.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    >Oui, le site est resté accessible sans interruption. Grâce au mécanisme de Rolling Update, Kubernetes conserve les anciens pods fonctionnels (nginx:1.28) en service tant que le nouveau pod n'a pas démarré avec succès, et le Service ne route le trafic que vers les pods prêts (Ready).

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    >https://humble-bassoon-xgww45p4xxpf6vgx-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    >Oui, Portainer et kubectl communiquent exactement avec le même composant central : l'API Server de Kubernetes (kube-apiserver) sur le nœud control-plane (k3d-lab-server-0). Portainer n'est qu'une interface graphique qui envoie des requêtes REST à l'API Server exactement comme le fait kubectl en ligne de commande — c'est pourquoi les ressources créées depuis Portainer (mario) sont immédiatement visibles via kubectl, et inversement (site, web).

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** ... / ...
