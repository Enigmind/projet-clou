# ***Projet Cloud***

### Problématique : 
On imgine qu'on a une grosse entreprise avec plusieurs sites à différents endroits du monde et on veut pouvoir déployer un service (une app) en utilisant ces mêmes sites en faisant un cluster des serveurs possédés.

### Introduction :

On a fait le projet en 2 parties. Tout d'abord, on a cherché à comprendre le fonctionnement de Rancher et on a "joué" avec afin de découvrir un maximum ses fonctionamités. Pour se faire, on a trouvé un vagrant qui déploie un cluster préconfiguré.

## Déploiement de Rancher avec Vagrant :

Nous avons tout d'abord cloné le répertoire git : 
```bash
git clone https://github.com/rancher/quickstart
```
On a un fichier VagrantFile qui exécute un *config.yaml* que l'on peut éditer pour créer autant de noeuds qu'on le souhaite et gérer leurs config (RAM, CPU etc.).

On est resté sur une config basique avec un serveur et un noeud : 
```yaml
default_password: admin
version: latest
kubernetes_version: v1.11.2-rancher1-1
ROS_version: 1.4.0
server:
  cpus: 1
  memory: 1500
node:
  count: 1
  cpus: 1
  memory: 1500
ip:
  master: 172.22.101.100
  server: 172.22.101.101
  node:   172.22.101.111
linked_clones: true
net:
  private_nic_type: 82545EM
  network_type: private_network
```
On éxecute ensuite le Vagrantfile avec :`vagrant up`

Une fois le serveur et le noeud déployé, on peut accéder à l'interface graphique de Rancher : 

![](https://i.imgur.com/bjhELr7.png)

Une fois l'interface et l'outil en main, on est passé à la partie plus technique du projet. On a voulu déployer rancher sur une machine distante (AWS) et déployer un cluster avec des machines toutes distantes.

## Déploiement de Rancher avec AWS

### Lancement de Rancher sur la première machine : 
On a essayé deux techniques pour déployer Rancher sur Aws : 

Tout d'abord nous avons récupéré le projet quickstart proposé sur le git de Rancher et suivi la doc.
On a renommé le fichier  *"terraform.tfvars.example"* en *"terraform.tfvars"*

Puis on l'a modifié pour y mettre la clé d'accès  AWS, la clé secrete, le nom de la clé ssh et le nombre de "nodes" : 
```bash
# Amazon AWS Access Key
aws_access_key = "your-aws-access-key"
# Amazon AWS Secret Key
aws_secret_key = "your-aws-secret-key"
# Amazon AWS Key Pair Name
ssh_key_name = "your-aws-key-pair-name"
# Region where resources should be created
region = "us-west-2"
# Resources will be prefixed with this to avoid clashing names
prefix = "myname"
# Admin password to access Rancher
admin_password = "admin"
# rancher/rancher image tag to use
rancher_version = "latest"
# Count of agent nodes with role all
count_agent_all_nodes = "1"
# Count of agent nodes with role etcd
count_agent_etcd_nodes = "0"
# Count of agent nodes with role controlplane
count_agent_controlplane_nodes = "0"
# Count of agent nodes with role worker
count_agent_worker_nodes = "0"
# Docker version of host running `rancher/rancher`
docker_version_server = "17.03"
# Docker version of host being added to a cluster (running `rancher/rancher-agent`)
docker_version_agent = "17.03"
# AWS Instance Type
type = "t3.medium"

```
On a ensuite lancé `terrafom init`  puis `terrafom apply`

La partie "terraform" n'aillant pas fonctionné nous avons décidé de le faire manuellement. Pour celà nous avons créé nous même une VM sur AWS, nous avons installé docker sur celle-ci ainsi qu'autorisé les ports en entrée et en sortie. Ensuite nous avons utilisé la commande : 
```bash
docker run -d --restart=unless-stopped   -p 80:80 -p 443:443   rancher/rancher:latest
```

Puis prendre l'url donné par le rancher et la coller dans un navigateur, le mot de passe est *admin*.

Une fois cela fait on accède à l'interface d'AWS et on peut y voir nos VMs (un manager et un worker) créer grâce à terraform. Du coup on peut accèder à notre interface Rancher via l'adresse IP : https://35.180.30.86

```bash
#arrêter tous les conteneurs
docker stop $(docker ps -q)

#supprimer tous les conteneurs
docker rm $(docker ps -a -q)
```


### Création du manager
On fait rejoindre une première machine dans le cluster en tant que "manager".
Pour ce faire, on va dans "edit cluster" puis "customize node run command" et on génère une commande avec les tags voulus pour faire rejoindre la machine : 
```bash
sudo docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run rancher/rancher-agent:v2.1.1 --server https://35.180.30.86 --token 7nkccd4rnwlkwl8bvbsrtzxnt2k8d29q5xjf68j4v9grpzpzwwvn8j --ca-checksum 3b20134607cbfa072d459b28fae387dcf0001ab5dbc551d9d0143cd09291480b --node-name manageryoann --address 18.234.27.117 --etcd --controlplane --worker
```
On rentre cette commande sur la machine que ayant l'IP '18.234.27.117' (il faut un démon docker actif) et on attend que rancher configure le manager.

### Faire rejoindre les Noeuds

Pour les noeuds, on fait globalement la même chose à part que l'on ne mettra pas les mêmes flags dans la commande. Sinon, c'est pareil, on utilise l'adrese IP publique de la machine et on génère la commande que l'on entrera sur la machine destinée à devenir "worker"

## Déploiement d'une app avec Rancher

On peut déployer toute sortes d'app. certaines sont répertoriées dans un catalogue. Pour le reste, il faut importer une image docker depuis docker hub (c'est ce qu'on a fait).

On ouvre un projet (default ou un qu'on a créé) et on clique sur workload puis sur 'Deploy'. On remplit les champs : le nom qu'on veut donner au contenenur, le nom de l'appli (avec le chemin sur docker hub) (exemple : rancher/hello-world ou enigmind/app). On peut choisir d'exposer un port afin que l'appli soit disponible lorsque l'on se connecte au noeud via l'IP publique ou créer un ingress après coup. On peut aussi choisir sur quel noeud déployer le conteneur.

Une fois l'app déployée, on peut choisir de créer un ingress pour que l'app soit accessible via sa propre adresse. Pour cela, il faut cliquer sur Ingress et remplir les champs demandés en renseignant le conteneur que l'on veut déployer.
