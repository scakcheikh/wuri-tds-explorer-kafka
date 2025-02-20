# **Travaux Dirigés (TD) : Exploration de Kafka avec Java**  

Ce TD a pour objectif de découvrir Apache Kafka en plusieurs étapes :  

1. **Étape 1** : Publication et consommation d’événements Kafka en Java.  
2. **Étape 2** : Utilisation de **Kafka Connect** pour connecter Kafka à une base de données.  
3. **Étape 3** : Utilisation de **Kafka Streams** pour le traitement en temps réel des données.  

---

## **🚀 Étape 1 : Publication et consommation d’événements Kafka avec Java**  

### **Objectif**  
- Configurer Kafka.  
- Créer un producteur Kafka en Java.  
- Créer un consommateur Kafka en Java.  

### **1. Installation et démarrage de Kafka**  

Téléchargez la dernière version de kafka et extraire
```tar -xzf kafka_2.13-3.9.0.tgz
cd kafka_2.13-3.9.0
```

Démarrer Kafka avec KRaft :  

Mode local:
```
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/kraft/reconfig-server.properties
bin/kafka-server-start.sh config/kraft/reconfig-server.properties
```

Avec Docker:
```
docker pull apache/kafka:3.9.0
docker run -p 9092:9092 apache/kafka:3.9.0
```
### **2. Création d’un topic**  
```sh
kafka-topics.sh --create --topic events --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1  
```

### **3. Producteur Kafka en Java**  
Créez une classe `KafkaProducerExample.java` :
```java
import org.apache.kafka.clients.producer.*;

import java.util.Properties;

public class KafkaProducerExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        KafkaProducer<String, String> producer = new KafkaProducer<>(props);

        for (int i = 0; i < 10; i++) {
            ProducerRecord<String, String> record = new ProducerRecord<>("events", "clé" + i, "message" + i);
            producer.send(record);
        }

        producer.close();
    }
}
```

### **4. Consommateur Kafka en Java**  
Créez une classe `KafkaConsumerExample.java` :
```java
import org.apache.kafka.clients.consumer.*;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class KafkaConsumerExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "test-group");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList("events"));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
            for (ConsumerRecord<String, String> record : records) {
                System.out.println("Reçu: " + record.key() + " -> " + record.value());
            }
        }
    }
}
```

### **5. Test**
- **Exécutez le producteur** → Il envoie 10 messages.  
- **Exécutez le consommateur** → Il doit afficher les messages reçus.

---

## **🛠 Étape 2 : Utilisation de Kafka Connect pour intégrer une base de données**  

### **Objectif**  
- Configurer **Kafka Connect** pour connecter Kafka à **PostgreSQL** (source) et à un fichier (sink).  

### **1. Installation de PostgreSQL**  
```sh
sudo apt update && sudo apt install postgresql
```
Créez une base de données et une table :  
```sh
psql -U postgres
CREATE DATABASE sales;
\c sales
CREATE TABLE orders (id SERIAL PRIMARY KEY, product VARCHAR(100), amount DECIMAL);
INSERT INTO orders (product, amount) VALUES ('Laptop', 1500), ('Phone', 800), ('Smartphone', 950), ('Screen', 2000);
```

### **2. Configuration de Kafka Connect (source PostgreSQL)**  
Ajoutez le connecteur Debezium PostgreSQL dans `config/connect-standalone.properties` :  
```properties
bootstrap.servers=localhost:9092
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
```

Ajoutez un fichier `config/postgres-source.properties` :
```properties
name=postgres-source
connector.class=io.debezium.connector.postgresql.PostgresConnector
database.hostname=localhost
database.port=5432
database.user=postgres
database.password=postgres
database.dbname=sales
database.server.name=pg-server
table.whitelist=public.orders
topic.prefix=sales
```

Lancez Kafka Connect :  
```sh
connect-standalone.sh config/connect-standalone.properties config/postgres-source.properties
```

Vérifiez que Kafka reçoit les événements :  
```sh
kafka-console-consumer.sh --topic sales.public.orders --bootstrap-server localhost:9092 --from-beginning
```

---

## **⚡ Étape 3 : Kafka Streams pour l’analyse en temps réel des commandes**  

### **Objectif**  
- Lire les commandes depuis Kafka.  
- Calculer le total des ventes par produit.  
- Publier les résultats dans un nouveau topic.  

### **1. Création du topic de sortie**  
```sh
kafka-topics.sh --create --topic sales-aggregated --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
```

### **2. Implémentation de Kafka Streams**
Ajoutez `kafka-streams` à `pom.xml` :
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.5.1</version>
</dependency>
```

Créez `KafkaStreamsExample.java` :
```java
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;

import java.util.Properties;

public class KafkaStreamsExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "sales-streams-app");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.Double().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        // Lecture des commandes depuis Kafka
        KStream<String, Double> salesStream = builder.stream("sales.public.orders",
                Consumed.with(Serdes.String(), Serdes.Double()));

        // Agrégation par produit
        KTable<String, Double> totalSales = salesStream
                .groupByKey()
                .reduce(Double::sum);

        // Écriture des résultats dans un topic
        totalSales.toStream().to("sales-aggregated", Produced.with(Serdes.String(), Serdes.Double()));

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### **3. Test**
- Insérez une nouvelle commande dans PostgreSQL :  
  ```sh
  INSERT INTO orders (product, amount) VALUES ('Tablet', 600);
  ```
- Vérifiez les résultats agrégés :  
  ```sh
  kafka-console-consumer.sh --topic sales-aggregated --bootstrap-server localhost:9092 --from-beginning
  ```

---

## **🎯 Conclusion**
Avec ce TD, vous avez :  
✅ Envoyé et consommé des messages Kafka en Java.  
✅ Connecté Kafka à une base de données avec Kafka Connect.  
✅ Utilisé Kafka Streams pour agréger des ventes en temps réel.  

🎯 **Extensions possibles** :  
- Déployer une stack pour surveiller le cluster kafka avec Prometheus et Grafana 🚀
