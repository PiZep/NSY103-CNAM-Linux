AUDITEUR: ZEPPA Pierre

# Question 1
**Processus clients et serveur BILLET sur une même machine, processus lourds.**
## Choix des outils
### Pour les requêtes client-server
Dans le cas qui nous concerne, tous les processus sont des processus lourds.
#### Consultation par le client
Il y a un seul *processus serveur Consultation*. Il faut qu'il sache quel client lui envoie une requête et renvoyer celle-ci de manière ciblée.
Les processus clients sont identifiés par leur PID. Les outils de communication les plus adaptés semble être les **MSQ**, où les types de chaque requête renvoyé par le *processus serveur Consultation* seront le PID du client. On pourrait également envisager l'utilisation des tubes nommés. Néanmoins, ils nécessiteraient la création de deux fichiers *tube* par requête, la communication avec un tube étant unilatérale.
#### Réservation par le(s) client(s)
Chaque réservation par un client se traduit par la création d'un *processus fils* du *processus serveur Réservation*.
Les **MSQ** sont également adaptés pour la requête de réservation. La file d'attente est créée par le *processus Réservation __Père__* et utilisée par chaque processus client Réservation mis en lien avec un *processus Réservation __Fils__*.
### Pour la requête de réservation, entre Père et Fils
#### Transfert de la requête
La requête de réservation est envoyée au *processus Réservation __Père__*. Ce dernier la transmet par **tube anonyme** au *processus Réservation __Fils__*.
#### Gestion de la mémoire
Les informations concernant les réservations sont stockées en mémoire centrale, dans une table, et le *processus Réservation* génère pour chaque requête de réservation un *processus fils*. Plusieurs *processus fils* doivent accéder à la table ; celle-ci est donc définit comme une zone de **mémoire partagée**.
Chaque processus fils accède à la table et peut potentiellement entrer en concurrence avec un autre *processus fils*. Un **groupe de sémaphores** est donc mis en place sur la table.

## Structure des messages échangés
**Postulats de départ:**
* Un `spectacle` est une structure, composée d'un `char\*` correspondant au nom du spectacle et d'un `int` correspondant au nombre de places restantes.
```c
struct spectacle {
	char *nom;
	int nbPlacesRestantes;
};
```
* Les accès des requêtes aux `spectacle`s sont effectués directement par leur indice dans la table. On suppose que celui-ci est déterminé via une structure intermédiaire de type table de hachage ou équivalent.
    * Cette structure intermédiaire n'est pas implémentée, par commodité (la mienne).
* Les clés des processus de communication sont codées en dur, et non générées via `ftok()`.
* La table des spectacles est générée 

### Consultation entre Client et BILLET
Le client envoie une requête dont la structure est la suivante :
```c
struct consultation {
	long type;  // type de la requête
	int id;     // indice du spectacle dans la table
	pid_t pid;  // pid du processus client, pour type de la réponse
};
```
En retour, le processus Consultation du serveur BILLET renvoie :
```c
struct reponse {
	long type;      // type de la requête
	int nbPlaces;   // nombre de places restantes
};
```
### Réservation entre Client et BILLET
La requête du client est similaire, avec l'ajout du nombre de places réservées :
```c
struct reservation {
	long type;      // type de la requête
	int id;         // indice du spectacle dans la table
	int nbPlace;    // nombre de places réservées
	pid_t pid;      // pid du processus client, pour type de la réponse
};
```
En réponse, le processus fils de Réservation envoie un message, soit d'acquittement, soit d'erreur :
```c
struct validation {
	long type;      // type de la requête
	char *Reponse;  // acquittement ou message d'erreur
};
```
### Réservation : communication entre Père et Fils
La requête de réservation envoyée par le client au *processus Réservation __Père__* est transmise via tube anonyme au *processus Réservation __Fils__*.

# Question 2
**Processus clients et serveur BILLET sur une même machine, processus légers.**
## Communication au sein du processus
Le *processus serveur BILLET* est constitué de threads. Il n'est donc plus nécessaire de mettre en place une zone de mémoire partagée puisque l'ensemble des threads ont accès aux mêmes ressource.
En revanche, l'utilisation des sémaphores restent nécessaire.
## Communication avec le(s) processus Client(s) et gestion des threads
Le thread (et la procédure adaptée) est généré en fonction du type de requête reçue par le *serveur BILLET*. La structure est sensiblement identique à celle des processus lourds en s'acquitant de la gestion de la mémoire partagée.

# Question 3
**Processus clients et serveur BILLET sur machines différentes, serveur composé de processus lourds**
## Outils de communication
Dans le cas de machines distantes, il est nécessaire de recourrir à l'utilisation des sockets. Les protocoles de communications sont les suivants :
* **Consultation** : UDP
* **Réservation** : TCP
La **Consultation** ne requiert pas de contrôle particulier des informations, aussi le mode *"déconnecté"* semble suffisant. Ce n'est pas le cas de la **Réservation** pour lequel des erreurs sur les paquets transmits pourraient aboutir à des valeurs incohérentes des nombres de places.
