
## **Mise en place d'un système Pub/Sub avec RabbitMQ et Java**

### **Objectifs**
- Comprendre le modèle de messagerie **Publish/Subscribe**.
- Configurer et utiliser RabbitMQ comme broker de messages.
- Implémenter un producteur (publisher) et un consommateur (subscriber) en Java.
- Explorer les concepts de **exchanges**, **queues**, et **bindings** dans RabbitMQ.

---

### **Prérequis**
- Connaissances de base en Java.
- Installation de RabbitMQ (local ou via Docker).
- Bibliothèque cliente RabbitMQ pour Java (`com.rabbitmq:amqp-client`).

---

### **Étapes du TD**

#### **1. Configuration de RabbitMQ**
1. **Installer RabbitMQ** :
   - En local: suivre les instructions https://www.rabbitmq.com/docs/download#installation-guides 
   - Avec Docker pour lancer un conteneur RabbitMQ :
     ```bash
     docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:4.0-management
     ```
   - Accédez à l'interface de gestion RabbitMQ à l'adresse `http://localhost:15672` (login: `guest`, password: `guest`).

2. **Créer un Exchange de type "Fanout"** :
   - Un exchange de type fanout diffuse les messages à toutes les files d'attente qui lui sont liées.
   - Dans l'interface RabbitMQ, créez un exchange de type **fanout** nommé `pubsub_exchange`.

---

#### **2. Implémentation en Java**

##### **2.1. Ajouter la dépendance RabbitMQ**
- Ajoutez la dépendance suivante à votre projet Maven ou Gradle :
  ```xml
  <!-- Pour Maven -->
  <dependency>
      <groupId>com.rabbitmq</groupId>
      <artifactId>amqp-client</artifactId>
      <version>5.16.0</version>
  </dependency>
  ```

##### **2.2. Créer un Publisher (Producteur)**
- Écrivez un programme Java pour publier des messages dans l'exchange `pubsub_exchange`.
  ```java
  import com.rabbitmq.client.ConnectionFactory;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.Channel;

  public class Publisher {
      private final static String EXCHANGE_NAME = "pubsub_exchange";

      public static void main(String[] args) throws Exception {
          ConnectionFactory factory = new ConnectionFactory();
          factory.setHost("localhost");
          try (Connection connection = factory.newConnection();
               Channel channel = connection.createChannel()) {
              // Déclarer l'exchange de type fanout
              channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

              String message = "Hello, Subscribers!";
              channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes("UTF-8"));
              System.out.println(" [x] Sent '" + message + "'");
          }
      }
  }
  ```

##### **2.3. Créer un Subscriber (Consommateur)**
- Écrivez un programme Java pour consommer les messages à partir d'une file d'attente liée à l'exchange `pubsub_exchange`.
  ```java
  import com.rabbitmq.client.*;

  public class Subscriber {
      private final static String EXCHANGE_NAME = "pubsub_exchange";

      public static void main(String[] argv) throws Exception {
          ConnectionFactory factory = new ConnectionFactory();
          factory.setHost("localhost");
          Connection connection = factory.newConnection();
          Channel channel = connection.createChannel();

          // Déclarer l'exchange de type fanout
          channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
          // Créer une file d'attente temporaire et la lier à l'exchange
          String queueName = channel.queueDeclare().getQueue();
          channel.queueBind(queueName, EXCHANGE_NAME, "");

          System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

          DeliverCallback deliverCallback = (consumerTag, delivery) -> {
              String message = new String(delivery.getBody(), "UTF-8");
              System.out.println(" [x] Received '" + message + "'");
          };
          channel.basicConsume(queueName, true, deliverCallback, consumerTag -> {});
      }
  }
  ```

---

#### **3. Tester le système Pub/Sub**
1. **Lancer plusieurs Subscribers** :
   - Exécutez plusieurs instances du programme `Subscriber` pour simuler plusieurs consommateurs.

2. **Lancer le Publisher** :
   - Exécutez le programme `Publisher` pour envoyer un message. Observez que tous les Subscribers reçoivent le message.

---

#### **4. Exercices supplémentaires**
1. **Modifier le type d'Exchange** :
   - Changez l'exchange pour utiliser un type **direct** ou **topic**. Modifiez les programmes pour tester le routage des messages en fonction des clés de routage.

2. **Gérer la persistance des messages** :
   - Configurez les messages pour qu'ils soient persistants en utilisant `deliveryMode(2)` dans le Publisher.

3. **Ajouter des logs et des métriques** :
   - Intégrez un système de logging (comme Log4j) et surveillez les métriques de RabbitMQ via l'interface de gestion.
- 

---

### **Résultats Attendus**
- Les étudiants doivent être capables de :
  - Configurer RabbitMQ et créer un exchange de type fanout.
  - Implémenter un producteur et un consommateur en Java.
  - Comprendre comment les messages sont diffusés dans un système Pub/Sub.
  - Explorer des concepts avancés comme les types d'exchanges et la persistance des messages.

---

### **Questions pour Réflexion**
1. Quelle est la différence entre un exchange de type **fanout** et un exchange de type **direct** ?
2. Comment RabbitMQ gère-t-il les consommateurs déconnectés dans un système Pub/Sub ?
3. Quels sont les avantages et les inconvénients de la persistance des messages ?

---

Ce TD offre une introduction pratique à RabbitMQ et au modèle Pub/Sub, tout en permettant aux étudiants d'explorer des concepts plus avancés.
