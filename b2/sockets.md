# Les sockets : communication bas niveau entre machines

## Introduction

- Objectif : écrire des programmes qui communiquent à travers un réseau
- Principe de base : communication directe par **IP + port**
- On se limite aux machines sur un **même LAN**
  - la communication entre IP de différents réseaux pose d'autres problèmes que l'on ne va pas traiter ici
  - voir par exemples les _websockets_ pour ceux que ça intéresse
- Le concept fondamental est celui du **modèle client/serveur** :
  - le programme serveur (que l'on confond souvent abusivement avec la machine qui l'accueille) fournit un « service » à d'autres programmes (les clients, qui tournent typiquement sur d'autres machines)
  - ex. : envoi de fichiers, de pages web, impression, communication inter-joueurs, calculs complexes distribués...
  - la distinction client/serveur est parfois floue : un client peut devenir serveur dans certaines situations, et inversement (dès que les communications entre les deux parties deviennent plus complexes qu'un serveur qui attend toujours le même type de requête et qui se contente de renvoyer une réponse définitive à cette requête)

## Les sockets

- Les **sockets** sont des programmes capables de transférer des données entre deux processus (éventuellement sur des machines différentes)
- C'est une communication _full-duplex_ : des données peuvent transiter de A vers et de B vers A, et ceci *simultanément*
- La communication inter-machines par sockets (TCP/IP) implique des traitements très bas niveau
- Mais Les sockets sont implémentés sous forme de classe en Java qui fournissent une **API (méthodes publiques) de haut niveau**
- Il y a deux types (et donc deux classes) de sockets :
  - La classe `Socket` a des méthodes qui permettent de :
    - se connecter à une machine distante (`connect`)
    - envoyer des données (via l'objet retourné par `getOutputStream`)
    - recevoir des données (via l'objet retourné par `getInputStream`)
    - fermer une connexion
  - La classe `ServerSocket` a des méthodes qui permettent de :
    - écouter sur un port spécifié et accepter une requête entrante (`accept`)
    - fermer une connexion
- Typiquement, le serveur aura :
  - un `ServerSocket` pour gérer les demandes de connexions entrantes
  - un `Socket` pour chaque client connecté (envoi/réception de données)
- Et chaque client aura :
  - un `Socket` pour gérer la connexion au serveur (envoi/réception de données)

## Les streams de données

- La bibliothèque de classes Java possède tout un tas de classes pour lire et écrire des flux de données (fichiers, réseaux...)
- Les objets `Socket` disposent de flux en entrée et en sortie
- la méthode `getInputStream` retourne un objet de type `InputStream` : c'est le flux entrant
- la méthode `getOutputStream` retourne un objet de type `OutputStream` : c'est le flux sortant
- Dans les exemples qui suivent, on va construire à partir de ces flux des objets de plus haut niveau :
  - un `DataInputStream` à partir du `InputStream`
  - un `DataOutputStream` à partir du `OutputStream`
  - ces deux classes sont plus faciles à manipuler que les deux classes de plus bas niveau

## Le serveur

Ci-dessous, un programme qui implémente un serveur simple :

- il attend les connexions sur le port spécifié
- Quand un client se connecte, le serveur s'attend à recevoir son adresse IP explicitement depuis le flux d'entrée
- Ensuite il va répéter ce traitement indéfiniment tant que le client est connecté :
  - reception de deux entiers depuis le client
  - calcul de la somme
  - envoi de la somme au client
- Quand le client est déconnecté, le serveur attend la connexion d'un nouveau client
- Notez comme la méthode `main` instancie un objet de sa propre classe avant d'appeler la méthode `demarrer` sur cette instance.
  - `demarrer` est une méthode d'instance (non statique), il faut donc disposer d'une instance pour l'appeler
  - ici il n'y a qu'un seul serveur, on aurait très bien pu mettre `demarrer` en statique (il aurait alors fallu que `serverSocket` et `port` soient aussi déclarés statiques)
- Plusieurs constructions `try/catch` requises pour ce genre de programme : beaucoup de problèmes réseaux peuvent évidemment se produire
  - ça « pollue » un peu le code mais elles sont obligatoires
  - => Java ne laissera pas des instructions potentiellement génératrices d'exceptions rester en dehors d'un `try/catch`

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;

public class SimpleServer {

  private ServerSocket serverSocket;
  private final int port = 8999;

  private void demarrer() {

    // Initialisation du serveur
    try {
      serverSocket = new ServerSocket(port);
    } catch (Exception ex) {
      System.err.println("!!! Erreur lors de la création du socket serveur.");
      System.err.println("!!! Détails : " + ex.getMessage());
      // Échec de création du serveur - Sortie anticipée du programme
      System.exit(0);
    }

    // Le socket serveur est démarré et prêt à recevoir des connexions.

    // Le while infini permet de répondre aux requêtes de connexion indéfiniment
    // (jusqu'à l'arrêt du serveur)
    while (true) {
      try {
        System.out.println("### Écoute sur le port " + port);
        // Attend une connexion (bloquant).
        // Retourne un objet Socket quand un client se connecte.
        Socket clientSocket = serverSocket.accept();
        System.out.println("### Connexion établie.");

        // On va maintenant demander à clientSocket de nous fournir des canaux
        // de communication en entrée et en sortie avec ce client

        // stream des données CLIENT => SERVEUR
        InputStream inStream = clientSocket.getInputStream();
        // Encapsule le stream précédent dans un objet de plus haut niveau
        // (plus facile à travailler)
        DataInputStream inDataStream = new DataInputStream(inStream);

        // stream des données SERVEUR => CLIENT
        OutputStream outStream = clientSocket.getOutputStream();
        // Encapsule le stream dans un objet de plus haut niveau
        DataOutputStream outDataStream = new DataOutputStream(outStream);

        // Attend une string sur le stream d'entrée (bloquant).
        // La string attendue est l'adresse du client.
        // Il faut que le client sache que le serveur attend cette info particulière
        // à ce moment-là (idem pour la suite des échanges)
        String ipClient = inDataStream.readUTF();
        System.out.println("--> Nouveau client à l'adresse : " + ipClient);

        // Le serveur va traiter les requêtes de ce client
        // tant que le client est connecté.
        // Une déconnexion (voulue ou non) va entraîner une exception qui
        // sortira de la boucle infinie pour aller dans le 'catch' et ensuite
        // revenir à la boucle while plus haut.
        try {
          while (true) {
            // Lit un entier depuis le client (bloquant)
            int nb1 = inDataStream.readInt();
            System.out.println("--> Premier nombre reçu : " + nb1);

            // Un autre (bloquant)
            int nb2 = inDataStream.readInt();
            System.out.println("--> Deuxième nombre reçu : " + nb2);

            // Ce serveur calcule juste la somme des deux entiers reçus
            int somme = nb1 + nb2;
            System.out.println("<-- Somme retournée : " + somme);

            // Envoi de la réponse au client
            outDataStream.writeInt(somme);

            System.out.println("### Traitement de cette requête terminée.");
          }
        } catch (IOException ex) {
          // connexion interrompue (en général car le client a terminé)
          System.out.println("### Le client est déconnecté");
        }
      } catch (IOException ex) {
        // connexion interrompue
        System.out.println("!!! La connexion a été interrompue.");
        System.err.println("!!! Erreur : " + ex.getMessage());
      }
    }
  }

  public static void main(String[] args) {
    SimpleServer serveur = new SimpleServer();
    serveur.demarrer();
  }
}
```

## Le client

```java
package fr.pgah.sockets;

import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;
import java.net.UnknownHostException;

public class SimpleClient {

  private final int portServeur = 8999;
  private String adresseServeur = "127.0.0.1";

  private void effectuerRequete() {

    try {
      System.out.println(
          "### Tentative de connexion à " + adresseServeur + ":" + portServeur + "...");
      // Lance une exception si serveur ne répond pas
      Socket connexionSocket = new Socket(adresseServeur, portServeur);
      System.out.println("### Connexion établie.");

      // Les streams d'entrée et de sortie
      InputStream inStream = connexionSocket.getInputStream();
      DataInputStream inDataStream = new DataInputStream(inStream);
      OutputStream outStream = connexionSocket.getOutputStream();
      DataOutputStream outDataStream = new DataOutputStream(outStream);

      // Envoi de notre adresse IP
      String ip = connexionSocket.getLocalAddress().getHostAddress();
      outDataStream.writeUTF(ip);

      // On envoie maintenant une série de 3 couples
      // d'entiers aléatoires au serveur
      for (int i = 0; i < 3; i++) {
        int entier1 = (int) (100 * Math.random());
        int entier2 = (int) (100 * Math.random());

        System.out.println("<-- Envoi nombre 1 : " + entier1);
        outDataStream.writeInt(entier1);
        System.out.println("<-- Envoi nombre 2 : " + entier2);
        outDataStream.writeInt(entier2);

        // Lit et affiche la réponse du serveur
        int res = inDataStream.readInt();
        System.out.println("--> Réponse serveur : " + res);
      }
      System.out.println("### Fermeture connexion...");
      connexionSocket.close();
      System.out.println("### Connexion terminée.");
    } catch (UnknownHostException ex) {
      System.err.println("!!! Hôte inconnu");
      System.err.println("!!! Détails : " + ex.getMessage());
      return;
    } catch (IOException ex) {
      System.err.println("!!! Erreur réseau");
      System.err.println("!!! Détails : " + ex.getMessage());
      return;
    }
  }

  public static void main(String[] args) {
    SimpleClient client = new SimpleClient();
    client.effectuerRequete();
  }
}
```

Rien de particulier si vous avez compris le code serveur. Ce client se connecte (à l'instanciation du `Socket`), envoie son IP, puis trois couples d'entiers aléatoires. Il termine ensuite la connexion proprement.

## Test

- Construisez deux projets Maven sur deux instances différentes de votre IDE : l'un pour le serveur, l'autre pour le client.
- Lancez d'abord le serveur, qui va attendre une connexion.
- Puis lancez le client.
- Vous devriez voir les échanges se faire entre les deux programmes.
- Le serveur reste à l'écoute tant que vous ne fermez pas le programme. Relancer le programme client plusieurs fois, le serveur répondra.
- Si vous disposez de plusieurs machines sur votre réseau local, vous pouvez faire le test entre deux machines différentes
  - remplacez alors `127.0.0.1` dans le programme client par l'adresse IP de la machine hôte serveur
  - veillez à ce que vos _firewalls_ autorisent les échanges si besoin
- Attention : cette méthode ne fonctionnera qu'au sein d'un même réseau.

## Exemple de sortie

Voici ce que ça donne côté serveur. Notez qu'on a dû terminer le programme avec `Ctrl-C`.

```
### Écoute sur le port 8999
### Connexion établie.
--> Nouveau client à l'adresse : 127.0.0.1
--> Premier nombre reçu : 84
--> Deuxième nombre reçu : 98
<-- Somme retournée : 182
### Traitement de cette requête terminée.
--> Premier nombre reçu : 28
--> Deuxième nombre reçu : 10
<-- Somme retournée : 38
### Traitement de cette requête terminée.
--> Premier nombre reçu : 89
--> Deuxième nombre reçu : 52
<-- Somme retournée : 141
### Traitement de cette requête terminée.
### Le client est déconnecté
### Écoute sur le port 8999
Terminer le programme de commandes (O/N) ? O
```

Et côté client :

```
### Tentative de connexion à 127.0.0.1:8999...
### Connexion établie.
<-- Envoi nombre 1 : 84
<-- Envoi nombre 2 : 98
--> Réponse serveur : 182
<-- Envoi nombre 1 : 28
<-- Envoi nombre 2 : 10
--> Réponse serveur : 38
<-- Envoi nombre 1 : 89
<-- Envoi nombre 2 : 52
--> Réponse serveur : 141
### Fermeture connexion...
### Connexion terminée.
```

## Serveur capable de répondre à plusieurs clients simultanément : introduction aux ***threads***

Dans la version précédente, le serveur ne peut accepter qu'une seule connexion à la fois : il doit traiter toutes les requêtes du client jusqu'à ce que celui-ci se déconnecte. Ensuite le serveur revient en début de boucle `while`, il est alors prêt à écouter de nouveau sur le port spécifié pour accepter une éventuelle nouvelle connexion.

Pour un serveur, il est très limitatif de ne pouvoir servir qu'un seul client à la fois. C'est pourquoi tous les serveurs ont en général la capacité d'accepter et de traiter plusieurs requêtes de la part de clients différents en même temps. À l'intérieur d'un même programme, cela se fait grâce au ***multithreading*** : plusieurs ***threads*** sont lancés au besoin. Ce sont comme des sous-programmes indépendants. Le temps processeur se divise entre tous les _threads_, et cela donne l'impression que tout se passe en même temps. C'est ce qui se passe sur tous les systèmes d'exploitation modernes, qui sont tous multi-tâches. Le fait que la machine dispose éventuellement de plusieurs processeurs implique que les _threads_ et processus disposent de plus de resources processeur disponibles, mais ça ne change pas vraiment le concept.

Notre nouveau serveur va toujours faire tourner dans une boucle infinie mais, cette fois, à chaque connexion d'un client, il ne va pas se dédier au traitement des requêtes de ce client. Il va instancier un nouveau thread qui va s'occuper de ce client tandis que le serveur se rend de nouveau immédiatement disponible pour accepter de nouveaux clients.

```java
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class MultithreadedServer {

  private static final int port = 8999;

  public static void main(String[] args) {
    ServerSocket serverSocket = null;

    // Instanciation du serveur
    try {
      serverSocket = new ServerSocket(port);
    } catch (Exception ex) {
      System.err.println("!!! Erreur lors de la création du socket serveur.");
      System.err.println("!!! Détails : " + ex.getMessage());
      // Échec de création du serveur - Sortie anticipée du programme
      System.exit(0);
    }

    try {
      // Boucle infinie
      while (true) {
        System.out.println("### Écoute sur le port " + port);
        Socket clientSocket = serverSocket.accept();

        // Cette fois-ci, au lieu de traiter lui-même la requête,
        // Le serveur instancie un nouveau thread qui va s'occuper
        // de ce client.
        // La classe AdditionServiceThread dérive de la classe 'Thread'
        // (c'est ce qui permet à l'objet 'threadPourClient' de s'exécuter
        // "en même temps" que les autres threads et que le serveur).
        AdditionServiceThread threadPourClient = new AdditionServiceThread(clientSocket);

        // Puis il reste à appeler la méthode 'start' sur ce thread.
        // Cette méthode ne fait pas partie de notre code, elle est "contenue"
        // dans la classe 'Thread' de laquelle dérive notre classe AdditionServiceThread.
        // Cette méthode se charge d'appeler la méthode 'run', que nous devrons
        // implémenter dans la classe dérivée.
        // CETTE APPEL N'EST PAS BLOQUANT : on démarre le thread mais le serveur
        // poursuit immédiatement son propre flux d'instructions.
        threadPourClient.start();
      }
    } catch (IOException ex) {
      System.err.println("!!! Erreur connexion.");
      System.err.println("!!! Détails : " + ex.getMessage());
    }
  }
}
```

Voyons maintenant la classe `AdditionServiceThread`. Un objet de cette classe est donc instancié à chaque nouvelle connexion d'un client. Cet objet sera exclusivement chargé de gérer les traitements qui correspondront à ce client précis via la méthode `run` (lancée automatiquement par l'appel à `start` dans le code serveur ci-dessus). Quand le client a terminé, la méthode `start` du thread se termine. Et quand `start` est terminé, ça veut dire que le thread a terminé son boulot : le thread est alors terminé, c'est-à-dire que le « sous-programme » ne s'exécute plus et tout l'espace mémoire qui lui a été alloué est libéré.

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;

// La classe dérive (extends) de la classe Thread de la JCL : elle va ainsi
// automatiquement disposer des fonctionnalités qui permettront à ses objets
// de s'exécuter en parallèle d'autres threads.
public class AdditionServiceThread extends Thread {

  // Ce compteur est STATIC : c'est une variable de classe, et ça veut dire qu'il
  // n'y en a qu'un, partagé entre tous les objets de cette classe.
  // On va s'en servir pour donner un identifiant unique (id) à chacun des threads
  // instanciés.
  private static int compteurDeConnexions;

  private int id;
  private final Socket socketClient;

  public AdditionServiceThread(Socket socketClient) {
    this.socketClient = socketClient;
  }

  @Override
  public void run() {
    try {
      compteurDeConnexions++;
      id = compteurDeConnexions; // cet id est unique
      System.out.println("### Connexion établie (id client : " + id + ").");

      // Canaux de communication entrant et sortant
      InputStream inStream = socketClient.getInputStream();
      DataInputStream inDataStream = new DataInputStream(inStream);
      OutputStream outStream = socketClient.getOutputStream();
      DataOutputStream outDataStream = new DataOutputStream(outStream);

      // On attend l'adresse IP du client (bloquant)
      String ipClient = inDataStream.readUTF();
      System.out.println("--> Adresse du client : " + ipClient);

      // Reçoit les entiers, calcule, et renvoie (tant que le client est connecté)
      while (true) {
        int nb1 = inDataStream.readInt();
        System.out.println("--> Premier nombre reçu depuis " + id + " : " + nb1);
        int nb2 = inDataStream.readInt();
        System.out.println("--> Deuxième nombre reçu depuis " + id + " : " + nb2);
        int somme = nb1 + nb2;
        System.out.println("<-- Somme retournée à client " + id + " : " + somme);
        outDataStream.writeInt(somme);
      }
    } catch (IOException ex) {
      System.out.println("### Le client " + id + " est déconnecté");
    }
  }
}
```

Le `compteurDeConnexion` est donc une variable statique : c'est une façon ici d'implémenter l'équivalent d'une clé auto-incrémentée dans une base de données relationnelle. À chaque fois qu'on instancie un objet `AdditionServiceThread` et que sa méthode `run` est appelée, on incrémente le compteur et on donne sa valeur actuelle à l'`id` du thread. Cela garantit que chaque thread a un identifiant unique.

L'annotation `@Override` indique que la méthode `run` est « dérivée » de la classe `Thread` : notre classe dérivée doit implémenter `run` pour indiquer quel est le code qui doit s'exécuter quand le thread démarre.

## Test Serveur multi-threadé

De nouveau, reproduisez le comportement observé ci-dessus sur votre propre machine. Vous ouvrirez une instance de VS Code avec le serveur qui tourne, et autant d'instance de VS Code supplémentaires que vous voulez de clients. Assurez-vous de bien comprendre tout ce qui se passe.

## Client multi-threadé

Ce serveur multi-threadé est immédiatement testable avec notre client existant, mais il est difficile de simuler des connexions concurrentes, même avec deux ordinateurs (le service est ici trop rapide à s'exécuter). On va donc le mettre un peu plus à l'épreuve en créant un client lui aussi multi-threadé qui va créer très rapidement une centaine de requêtes sur notre serveur. Le principe est le même que pour la création des threads du serveur.

```java
public class MultithreadedClients {

  private static final int portServeur = 8999;
  private static String adresseServeur = "127.0.0.1";
  private static final int nbClients = 100;

  public static void main(String[] args) {
    // Création des clients.
    // Les appels à 'start' sont non-bloquants, la création de tous les threads est donc
    // très rapide et ils "tapent" chacun immédiatement le serveur.
    for (int i = 0; i < nbClients; i++) {
      AdditionClientThread thread = new AdditionClientThread(adresseServeur, portServeur);
      thread.start();
    }
  }
}
```

Au passage, ce sont les mêmes concepts qui permettent les attaques de type DoS (déni de service), sauf que ce type d'attaque se fait aujourd'hui depuis plusieurs machines, en plus d'être multi-threadées (DDoS - *Distributed Denial of Service*).

Et voici l'implémentation de la classe qui représente un thread qui va faire une requête d'addition. Elle est semblable au code de notre client précédent, encapsulé dans la méthode `run` (sauf que cette fois chaque client ne fait plus qu'une requête d'addition au lieu de trois).

```java
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;
import java.net.UnknownHostException;

public class AdditionClientThread extends Thread {

  private final String adresseServeur;
  private final int portServeur;

  public AdditionClientThread(String adresseServeur, int portserveur) {
    this.adresseServeur = adresseServeur;
    this.portServeur = portserveur;
  }

  @Override
  public void run() {
    try {
      System.out.println(
          "### Tentative de connexion à " + adresseServeur + ":" + portServeur + "...");
      Socket connexionSocket = new Socket(adresseServeur, portServeur);
      System.out.println("### Connexion établie.");

      InputStream inStream = connexionSocket.getInputStream();
      DataInputStream inDataStream = new DataInputStream(inStream);
      OutputStream outStream = connexionSocket.getOutputStream();
      DataOutputStream outDataStream = new DataOutputStream(outStream);

      String ip = connexionSocket.getLocalAddress().getHostAddress();
      outDataStream.writeUTF(ip);

      int entier1 = (int) (100 * Math.random());
      int entier2 = (int) (100 * Math.random());

      System.out.println("<-- Envoi nombre 1 : " + entier1);
      outDataStream.writeInt(entier1);
      System.out.println("<-- Envoi nombre 2 : " + entier2);
      outDataStream.writeInt(entier2);

      int res = inDataStream.readInt();
      System.out.println("--> Réponse serveur : " + entier1 + " + " + entier2 + " = " + res);

      System.out.println("### Fermeture connexion...");
      connexionSocket.close();
      System.out.println("### Connexion terminée.");
    } catch (UnknownHostException ex) {
      System.err.println("!!! Hôte inconnu");
      System.err.println("!!! Détails : " + ex.getMessage());
      return;
    } catch (IOException ex) {
      System.err.println("!!! Erreur réseau");
      System.err.println("!!! Détails : " + ex.getMessage());
      return;
    }
  }
}
```

Si maintenant on lance notre `MultithreadedServer` puis notre `MultithreadedClients`, la sortie de chaque côté sera un peu chaotique.

- Côté serveur (extrait) :

```
...
--> Deuxième nombre reçu depuis 6 : 59
--> Premier nombre reçu depuis 16 : 4
--> Premier nombre reçu depuis 15 : 87
--> Premier nombre reçu depuis 14 : 55
### Connexion établie (id client : 20).
### Écoute sur le port 8999
--> Adresse du client : 127.0.0.1
<-- Somme retournée à client 6 : 133
--> Premier nombre reçu depuis 1 : 98
--> Premier nombre reçu depuis 5 : 55
### Connexion établie (id client : 21).
--> Premier nombre reçu depuis 20 : 32
### Écoute sur le port 8999
--> Adresse du client : 127.0.0.1
--> Premier nombre reçu depuis 21 : 9
### Écoute sur le port 8999
### Connexion établie (id client : 22).
### Écoute sur le port 8999
--> Adresse du client : 127.0.0.1
### Connexion établie (id client : 23).
### Connexion établie (id client : 24).
--> Premier nombre reçu depuis 22 : 85
...
```

- Côté client (extrait) :

```
...
<-- Envoi nombre 2 : 30
<-- Envoi nombre 2 : 79
<-- Envoi nombre 2 : 86
<-- Envoi nombre 2 : 6
--> Réponse serveur : 71 + 91 = 162
### Fermeture connexion...
--> Réponse serveur : 63 + 47 = 110
### Fermeture connexion...
--> Réponse serveur : 32 + 44 = 76
### Fermeture connexion...
### Fermeture connexion...
--> Réponse serveur : 44 + 38 = 82
### Fermeture connexion...
### Connexion terminée.
### Connexion terminée.
--> Réponse serveur : 31 + 73 = 104
### Fermeture connexion...
### Connexion terminée.
### Connexion terminée.
--> Réponse serveur : 33 + 82 = 115
...
```

Chaque thread travaille indépendamment des autres, et les sorties sur console se font de manière totalement désordonnée. On voit aussi qu'il se peut très bien que des threads ayant démarré avant finissent leur job *après* d'autres threads pourtant créés plus tardivement : on n'a aucune garantie que les requêtes soient traitées dans l'ordre d'arrivée, mais ce n'est pas ce qui nous intéresse ici. Ce qu'on veut, c'est traiter les clients en parallèle, et que chacun d'entre eux ait bien la réponse correcte par rapport à leur requête (on peut vérifier que c'est bien le cas en voyant le résultat correct des additions côté client).

## Test Clients et serveurs multi-threadés

 De nouveau, reproduisez le comportement. Ouvrez une instance de VS Code pour le client et une autre pour le serveur. Démarrez le serveur, puis le client, et observez. Vous pouvez varier le nombre de clients générés dynamiquement.

## TP : Appli de chat

Il vous est demandé d'écrire une simple application de chat sur ce modèle client/serveur. Appuyez-vous sur le code existant. Vous aurez un serveur qui gérera toutes les communications. Les clients se connecteront au serveur et enverront des messages (avec un identifiant permettant de reconnaître le posteur). Clients et serveurs devront donc tous les deux à la fois « écouter » des messages qui arrivent et envoyer des messages :

- Le serveur reçoit les messages de tous les clients
- Et il les *broadcast* (envoie à tous les clients)
- Un client reçoit et affiche tous les messages provenant du serveur (y compris ce qu'il a lui-même envoyés, qu'il faudra donc ignorer)
- Et il envoie les messages de l'opérateur local
