# TP Infrastructure as Code — Terraform et Docker

Ce dépôt contient un TP pour déployer un serveur Nginx local avec Terraform et Docker.

## Prérequis

- Terraform 1.5 ou plus récent
- Docker Desktop ou Docker Engine en cours d'exécution
- `make` (facultatif, pour les commandes raccourcies)

## Démarrage rapide

Depuis la racine du dépôt :

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
terraform apply
```

Après confirmation, ouvrez <http://localhost:8080>. La page servie vient de `html/` grâce à un montage bind ; ses modifications sont visibles sans reconstruire l'image.

Pour afficher les informations de déploiement ou supprimer les ressources :

```bash
terraform output
terraform destroy
```

Les mêmes opérations courantes sont disponibles via `make init`, `make fmt`, `make validate`, `make plan`, `make apply` et `make destroy`.

## Configuration

Les variables `container_name` et `external_port` sont définies dans [variables.tf](./variables.tf). Leurs valeurs par défaut sont `tp-iac-nginx` et `8080`.

Pour tester une autre configuration, créez un fichier de variables `dev.tfvars`, puis lancez :

```bash
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```

---
# Réponses :

## 9. Personnaliser la page Web

Question : quelle est la différence entre un montage bind et la copie d'un fichier dans une image Docker ?

Avec un montage bind, le conteneur accède directement au dossier de l'hôte : les modifications des fichiers locaux sont visibles sans reconstruire l'image. 
    Avec l'instruction copy, les fichiers sont intégrés à l'image au moment de sa construction (les modifications sur l'hôte ne sont donc pas répercutées).

---

## 10. Créer un réseau Docker dédié

- expliquez pourquoi le conteneur est connecté à ce réseau ;

    Le conteneur `tp-iac-nginx` est connecté au réseau `tp-iac-nginx-network` parce que Terraform le rattache à ce réseau dans sa configuration. 
    Docker lui attribue automatiquement une adresse IP disponible dans le sous-réseau, ici 172.19.0.2.

- indiquez l'intérêt de déclarer le réseau dans Terraform ;

    Déclarer le réseau dans Terraform permet de gérer sa création et sa suppression avec le reste de l’infrastructure, et de reproduire la même configuration de manière cohérente.

- vérifiez que le réseau est supprimé lors de la destruction du projet.
```bash
 % docker network inspect tp-iac-nginx-network
[]
```
---

## 11. Limiter l'exposition du service

Question : quelle est la différence entre 127.0.0.1 et une écoute sur toutes les interfaces ?
    
127.0.0.1 correspond à l’interface locale, le service est accessible uniquement depuis la machine qui l’héberge. 
    À l’inverse, écouter sur toutes les interfaces rend le port accessible via toutes les interfaces réseau de l’hôte, éventuellement depuis d’autres machines.

---

## 13. Inspecter l'état Terraform

1. Quelles ressources apparaissent dans terraform state list ?
    
    `terraform state list` affiche les ressources gérées par Terraform : 
    `docker_container.web`, `docker_image.nginx` et `docker_network.app_network`.


2. Quelles informations affiche terraform state show ?

    terraform state show docker_container.web affiche les détails connus du conteneur, notamment son nom, son image, son état d’exécution, son réseau et son adresse IP, son montage de fichiers ainsi que le port publié (127.0.0.1:8081 vers le port 80 du conteneur).


3. Pourquoi l'état Terraform doit-il être protégé ?

    L’état Terraform doit être protégé, car il contient des informations détaillées sur l’infrastructure et peut inclure des données sensibles. 


4. Pourquoi faut-il éviter de publier cet état dans un dépôt public ?

    Il faut éviter de publier cet état dans un dépôt public, car tout le monde pourrait consulter ces informations sensibles et la configuration de l’infrastructure.

---

## 15. Versionner avec Git

- Expliquez comment l'historique Git facilite la revue, la collaboration et le retour sur une modification.

L’historique Git facilite la revue en montrant précisément les modifications avec git diff. Ici, le changement commité ajoute une ligne vide dans `main.tf`.
Les commits permettent aussi aux autres de comprendre et de suivre les changements du projet.
Si une modification pose problème, Git permet de retrouver une version antérieure ou d’annuler le commit.

---

## 19. Exercices complémentaires


Exercice 1 — Modifier le nom
Expliquez les changements prévus.

```bash
Plan: 2 to add, 0 to change, 2 to destroy.

Changes to Outputs:
  ~ container_name = "tp-iac-nginx" -> "tp-iac-web"
  ~ network_name   = "tp-iac-nginx-network" -> "tp-iac-web-network"
```

Exercice 4 — Comprendre l'état

- Listez les ressources Terraform.

```bash
% terraform state list
docker_container.web
docker_image.nginx
docker_network.app_network
```

- Identifiez les dépendances entre l'image, le réseau et le conteneur.

    Le conteneur dépend de l'image (`docker_image.nginx.image_id`) et du réseau (`docker_network.app_network.name`). L'image et le réseau sont indépendants.

- Expliquez pourquoi le conteneur dépend de l'image et du réseau.

    Le conteneur est créé à partir de l'image et rattaché au réseau : les deux doivent donc exister avant lui. Terraform les crée en premier et les détruit en dernier.


Exercice 5 — Détecter une dérive

```bash
% docker stop tp-iac-web
% terraform plan
Plan: 1 to add, 0 to change, 0 to destroy.
```

Terraform détecte que le conteneur n'est plus actif et propose de le recréer. `terraform apply` restaure l'infrastructure.


Exercice 6 — CI/CD

```bash
Error: Unclosed configuration block
│ 
│   on main.tf line 23, in resource "docker_container" "web":
│   23: resource "docker_container" "web" {
│ 
│ There is no closing brace for this block before the end of the file. This
│ may be caused by incorrect brace nesting elsewhere in this file.
```

Exercice 7 — Nettoyage

```bash
% terraform plan -destroy
Plan: 0 to add, 0 to change, 3 to destroy.
% terraform destroy
% docker ps -a --filter name=tp-iac-web
% docker network inspect network
[]
```

Il ne faut pas supprimer l'état à la main : Terraform perdrait le lien avec les ressources, qui resteraient actives dans Docker sans pouvoir être détruites par `terraform destroy`.

---

## 20. Questions de compréhension


1. Quelle est la différence entre `terraform init`, `terraform plan` et `terraform apply` ?

   `terraform init` initialise le projet, `terraform plan` montre les changements prévus et `terraform apply` les applique.

2. Quel est le rôle d'un provider Terraform ?

   Un provider permet à Terraform de communiquer avec un service, comme Docker ou AWS, et d’y gérer des ressources.

3. À quoi sert le fichier `terraform.tfstate` ?

   `terraform.tfstate` conserve l’état connu des ressources gérées par Terraform et les relie à la configuration.

4. Pourquoi utilise-t-on des variables et des outputs ?

   Les variables permettent de paramétrer et de réutiliser une configuration selon l’environnement, sans modifier le code. Les outputs affichent des informations utiles après le déploiement, comme le nom du conteneur ou l’URL du service.

5. Qu'est-ce qu'une dérive de configuration ?

   Une dérive de configuration est une différence entre l’infrastructure décrite dans le code et son état réel.

6. Quelle différence existe-t-il entre Infrastructure as Code et configuration manuelle ?

   L’IaC décrit l’infrastructure dans des fichiers, ce qui permet de la reproduire. La configuration manuelle se fait directement, étape par étape.

7. Quel est l'intérêt de Git dans un projet IaC ?

   Git conserve l’historique des modifications et permet de comparer les versions pour relire et vérifier les changements. Les branches et les commits facilitent le travail à plusieurs, tandis que l’historique permet de retrouver ou de restaurer une version précédente en cas de problème.

8. Quelle est la différence entre un contrôle CI et un déploiement CD ?

   La CI (intégration continue) exécute automatiquement des vérifications sur les changements, par exemple le formatage, la validation et la génération d’un plan Terraform. La CD (déploiement continu) automatise la livraison ou l’application des changements dans un environnement. La CI vérifie donc le code ; la CD déploie l’infrastructure.

9. Pourquoi faut-il examiner le plan avant d'appliquer une modification ?

   Examiner le plan permet de comprendre les effets d’une modification et de repérer une action inattendue ou destructive avant de l’appliquer.

10. Quelles précautions faut-il prendre avec les fichiers d'état et les secrets ?

    Il faut protéger les fichiers d’état, éviter de publier les secrets et les exclure du dépôt Git.
