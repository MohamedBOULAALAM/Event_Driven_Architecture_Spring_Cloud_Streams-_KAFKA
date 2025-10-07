# Activité Pratique N°1 – Architecture Orientée Événements avec Kafka

> **Module** : Systèmes Parallèles et Distribués  
> **Technologies utilisées** : Apache Kafka, Docker, Spring Boot, Spring Cloud Streams, Kafka Streams, HTML/JavaScript (SSE)

---

## Introduction

Cette activité pratique vise à explorer les fondements de l’**architecture orientée événements (Event-Driven Architecture)** à travers l’utilisation d’**Apache Kafka**, une plateforme de streaming distribuée très populaire. L’objectif est de comprendre comment produire, consommer et traiter en temps réel des flux d’événements dans un système distribué.

Nous mettons en œuvre plusieurs composants :
- Un **producteur Kafka** exposé via une API REST,
- Un **consommateur Kafka** simple,
- Un **fournisseur (Supplier)** générant des événements simulés,
- Un **moteur d’analyse en temps réel** avec **Kafka Streams**,
- Une **interface web dynamique** affichant les résultats via Server-Sent Events (SSE).

L’environnement Kafka est déployé à la fois **manuellement** et via **Docker**, afin de comparer les deux approches.

---
## 1. Installation et test manuel de Kafka (sans Docker)

### Objectif
Valider le fonctionnement de base de Kafka en local, sans conteneurisation.

### Étapes réalisées

#### 1. Téléchargement et extraction
- Télécharge la version **binaire** depuis [https://kafka.apache.org/downloads](https://kafka.apache.org/downloads) (ex : `kafka_2.13-3.7.0.tgz`).
- **Extraction** :  
  Tu peux utiliser **7-Zip**, **WinRAR**, ou PowerShell :
  ```powershell
  tar -xzf kafka_2.13-3.7.0.tgz
  ```
- Puis accède au dossier :
  ```cmd
  cd kafka_2.13-3.7.0
  ```
#### 2. Démarrer ZooKeeper (nécessaire pour Kafka < 3.3)
```cmd
bin\windows\zookeeper-server-start.bat config\zookeeper.properties
```
>  Garde cette fenêtre ouverte (ne pas fermer le terminal).
#### 3. Démarrer le serveur Kafka
Dans une **nouvelle fenêtre CMD/PowerShell** :
```cmd
bin\windows\kafka-server-start.bat config\server.properties
```
> ! Assurer que ZooKeeper est toujours en cours d’exécution.
#### 4. Créer un topic
Dans une **troisième fenêtre** :
```cmd
bin\windows\kafka-topics.bat --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```
> À partir de Kafka 2.2+, on utilise `--bootstrap-server` au lieu de `--zookeeper`.
#### 5. Lancer le producteur (console producer)
```cmd
bin\windows\kafka-console-producer.bat --topic test-topic --bootstrap-server localhost:9092
```
> Tape quelques messages (un par ligne), puis **Ctrl+C** pour quitter.
#### 6. Lancer le consommateur (console consumer)
```cmd
bin\windows\kafka-console-consumer.bat --topic test-topic --from-beginning --bootstrap-server localhost:9092
```
> L'apparaître de tous les messages qu'on a saisis dans le producteur.

**Résultat** : Les messages saisis dans le producteur apparaissent immédiatement dans le consommateur, confirmant le bon fonctionnement de Kafka en mode local.

---
## 2. Utilisation de Kafka avec Docker

### Objectif
Reproduire la même configuration dans un environnement conteneurisé, plus proche des déploiements réels.

### Fichier `docker-compose.yml`

```yaml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

### Démarrer les conteneurs :
```bash
> docker-compose up -d
```

### Tester avec les outils Kafka (dans le conteneur Kafka) :
#### Commandes Kafka exécutées dans un conteneur Docker

> **Contexte** :  
> Travailler avec un cluster Kafka déployé via Docker. Le conteneur Kafka porte le nom `bdcc-kafla-broker` (note : probablement une coquille pour `bdcc-kafka-broker`), et le service Kafka à l’intérieur du réseau Docker est accessible via le hostname `broker` sur le port `9092`.
##### 1. **Consommer les messages du topic `R2`**
```bash
docker exec --interactive --tty bdcc-kafla-broker kafka-console-consumer --bootstrap-server broker:9092 --topic R2
```
- **Objectif** : Lire en temps réel tous les messages publiés dans le topic `R2`.
- **Comportement** :
  - Le consommateur se connecte au broker Kafka (`broker:9092`).
  - Il affiche **uniquement les valeurs** des messages (pas les clés).
  - Il **ne lit pas les messages historiques** (seulement ceux envoyés **après** le démarrage du consommateur), sauf si l’option `--from-beginning` est ajoutée.
- **Cas d’usage** : Vérifier qu’un producteur envoie bien des données dans `R2`.

##### 2. **Produire des messages dans le topic `R2`**
```bash
docker exec --interactive --tty bdcc-kafla-broker kafka-console-producer --bootstrap-server broker:9092 --topic R2
```
- **Objectif** : Envoyer manuellement des messages dans le topic `R2`.
- **Fonctionnement** :
  - Chaque ligne tapée dans le terminal est traitée comme un **message** (valeur seule, clé = `null`).
  - Le producteur reste actif jusqu’à ce que tu appuies sur **Ctrl+C**.
- **Cas d’usage** : Tester rapidement un topic sans écrire de code.

##### 3. **Consommer le topic `R66` avec affichage des clés et valeurs (format texte)**
```bash
docker exec --interactive --tty bdcc-kafla-broker kafka-console-consumer \
  --bootstrap-server broker:9092 \
  --topic R66 \
  --property print.key=true \
  --property print.value=true
```
- **Objectif** : Afficher **à la fois la clé et la valeur** de chaque message.
- **Format** :  
  Par défaut, Kafka utilise des **sérialiseurs/désérialiseurs de chaînes de caractères** (`StringDeserializer`).
- **Sortie exemple** :
  ```
  key1    valeur1
  user123 {"action":"login"}
  ```
- **Cas d’usage** : Déboguer un flux où la **clé** a une signification (ex: ID utilisateur, nom de page).

##### 4. **Consommer `R66` avec désérialisation explicite (clé = String, valeur = Long)**
```bash
docker exec --interactive --tty bdcc-kafla-broker kafka-console-consumer \
  --bootstrap-server broker:9092 \
  --topic R66 \
  --property print.key=true \
  --property print.value=true \
  --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
  --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```
- **Objectif** : Lire correctement des messages dont la **valeur est un entier long** (ex: compteur, timestamp).
- **Pourquoi ?**  
  Si les données ont été produites avec un `LongSerializer`, les lire comme du texte afficherait des caractères illisibles (octets bruts). Cette commande **interprète correctement** les bytes comme un `long`.
- **Exemple de sortie** :
  ```
  P1    42
  P2    128
  ```
- **Cas d’usage** : Consommer des agrégats (ex: résultats de Kafka Streams comme des compteurs).

##### 5. **Lister tous les topics Kafka**
```bash
docker exec --interactive --tty bdcc-kafla-broker kafka-topics --bootstrap-server broker:9092 --list
```
- **Objectif** : Afficher la liste complète des topics existants dans le cluster.
- **Utilité** :
  - Vérifier qu’un topic a bien été créé.
  - Découvrir les topics internes (ex: `__consumer_offsets`, `count-store-*` créés par Kafka Streams).
- **Sortie typique** :
  ```
  R2
  R66
  __consumer_offsets
  count-store-repartition
  ```

#### Remarques importantes

- Le nom du conteneur (`bdcc-kafla-broker`) doit correspondre à celui défini dans ton `docker-compose.yml`.
- L’hostname `broker` suppose que tu utilises un **réseau Docker personnalisé** (ex: défini dans `docker-compose`) où le service Kafka est nommé `broker`.
- Les options `--interactive` (`-i`) et `--tty` (`-t`) permettent d’avoir un terminal fonctionnel (saisie, historique, etc.).

#### Bonnes pratiques

- Pour les topics avec **données binaires** (Long, Integer, JSON, Avro), **toujours spécifier les désérialiseurs**.
- Utilise `--from-beginning` si tu veux lire **tous les messages existants**, pas seulement les nouveaux.
- En développement, préfère les outils comme **Conduktor**, **Offset Explorer**, ou **kcat** pour une meilleure UX.


> Ces commandes sont essentielles pour **tester, déboguer et valider** le comportement de ton architecture orientée événements avec Kafka.

![0.png](captures/img1.png)






---
## 3. Développement avec Spring Cloud Streams et Kafka Streams
Nous utilisons **Spring Boot** avec **Spring Cloud Stream** pour interagir facilement avec Kafka, et **Kafka Streams** pour le traitement en temps réel.
### Modèle de données
```java
public record PageEvent(String name, String user, Date date, long duration) {}
```
Il représente un événement de navigation sur une page web (nom de la page, utilisateur, horodatage, durée de visite).
---
### 3.1. Service Producer Kafka (via REST Controller)
Un endpoint REST permet de publier manuellement des événements.
```java
@RestController
public class PageEventController {
    @Autowired
    private StreamBridge streamBridge;
    
    @GetMapping("/publish")
    public PageEvent publish(String name, String topic) {
        PageEvent event = new PageEvent(
            name,
            Math.random() > 0.5 ? "U1" : "U2",
            new Date(),
            10 + new Random().nextInt(10000)
        );
        streamBridge.send(topic, event);
        return event;
    }
}
```
**Explication** :
- L’endpoint `/publish?name=P1&topic=page-events` crée un événement aléatoire.
- `StreamBridge` permet d’envoyer dynamiquement vers n’importe quel topic.

![Capture du producteur](captures/img.png)

---

### 3.2. Service Consumer Kafka
Consomme les événements et les affiche dans la console.
```java
@Component
public class PageEventHandler {
    @Bean
    public Consumer<PageEvent> pageEventConsumer() {
        return input -> {
            System.out.println("*****************");
            System.out.println(input.toString());
            System.out.println("*****************");
        };
    }
}
```
**Explication** :
- Spring Cloud Stream détecte automatiquement ce `Consumer<PageEvent>`.
- Il s’abonne au topic configuré (ex : `page-events`).

![Capture du consommateur](captures/img1.png)

---

### 3.3. Service Supplier Kafka
Un **Supplier** émet des messages de façon programmatique (ex : heartbeat, données simulées).
Alors, génère automatiquement des événements toutes les secondes.
```java
@Bean
public Supplier<PageEvent> pageEventSupplier() {
    return () -> new PageEvent(
        Math.random() > 0.5 ? "P1" : "P2",
        Math.random() > 0.5 ? "U1" : "U2",
        new Date(),
        10 + new Random().nextInt(10000)
    );
}
```
> ⏱️ Par défaut, Spring appelle le `Supplier` toutes les secondes (configurable via `spring.cloud.stream.bindings.pageEventSupplier-out-0.producer.poller.fixed-delay=10000 // par defaut`).
**Explication** :
- Le `Supplier` est invoqué périodiquement par Spring.
- Utile pour simuler un flux continu (ex: capteurs, logs, etc.).

![Capture du supplier](captures/img2.png)

---

### 3.4. Service de Data Analytics en temps réel avec Kafka Streams
Analyse en fenêtre glissante de 5 secondes le nombre de visites par page. On ajoute la fonction _kStreamFunction_ à la classe : _**PageEventHandler**_ :
```java
@Bean
public Function<KStream<String, PageEvent>, KStream<String, Long>> kStreamFunction() {
    return input ->
        input
            .map((k, v) -> new KeyValue<>(v.name(), v.duration()))
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Long()))
            .windowedBy(TimeWindows.of(Duration.ofSeconds(5)))
            .count(Materialized.as("count-store"))
            .toStream()
            .peek((k, v) -> {
                System.out.println("=======>  " + k.key());
                System.out.println("=======>  " + v);
            })
            .map((k, v) -> new KeyValue<>(k.key(), v));
}
```
**Explication** :
- Le flux est agrégé par nom de page (`P1`, `P2`).
- Une fenêtre de 5 secondes permet de compter les événements récents.
- Le store `"count-store"` est exposé via **Interactive Queries**.


### Interface Web en temps réel
Un endpoint SSE (`/analytics`) expose les agrégats toutes les secondes, on ajoute la fonction _analytics_ à la classe :  _**PageEventController**_ :
```java
@Autowired
private InteractiveQueryService interactiveQueryService;

@GetMapping(path = "/analytics",produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Map<String, Long>> analytics(){
   return Flux.interval(Duration.ofSeconds(1))
           .map(sequence->{
              Map<String,Long> stringLongMap=new HashMap<>();
              ReadOnlyWindowStore<String, Long> windowStore = interactiveQueryService.getQueryableStore("count-store", QueryableStoreTypes.windowStore());
              Instant now=Instant.now();
              Instant from=now.minusMillis(5000);
              KeyValueIterator<Windowed<String>, Long> fetchAll = windowStore.fetchAll(from, now);
              //WindowStoreIterator<Long> fetchAll = windowStore.fetch(page, from, now);
              while (fetchAll.hasNext()){
                 KeyValue<Windowed<String>, Long> next = fetchAll.next();
                 stringLongMap.put(next.key.key(),next.value);
              }
              return stringLongMap;
           });
}
```
Et une page HTML affiche les données en temps réel avec **Smoothie.js** :
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Analytics</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/smoothie/1.34.0/smoothie.min.js"></script>
</head>
<body>

<canvas id="chart2" width="300" height="200"></canvas>
<script>
  var index=-1;
  randomColor = function() {
    ++index;
    if (index >= colors.length) index = 0; return colors[index];
  }
  var pages=["P1","P2"];
  var colors=[
    { sroke : 'rgba(0, 255, 0, 1)', fill : 'rgba(0, 255, 0, 0.2)' },
    { sroke : 'rgba(255, 0, 0, 1)', fill : 'rgba(255, 0, 0, 0.2)'}
  ];
  var courbe = [];
  var smoothieChart = new SmoothieChart({tooltip: true});
  smoothieChart.streamTo(document.getElementById("chart2"), 500);
  pages.forEach(function(v){
    courbe[v]=new TimeSeries();
    col = randomColor();
    smoothieChart.addTimeSeries(courbe[v], {strokeStyle : col.sroke, fillStyle : col.fill, lineWidth : 2
    });
  });
  var stockEventSource= new EventSource("/analytics");
  stockEventSource.addEventListener("message", function (event) {
    pages.forEach(function(v){
      val=JSON.parse(event.data)[v];
      courbe[v].append(new Date().getTime(),val);
    });
  });

</script>
</body>
</html>
</body>
</html>
```
**Explication** :
- Le navigateur s’abonne au flux SSE.
- Chaque seconde, il reçoit un objet JSON avec les compteurs.
- Les courbes sont mises à jour en temps réel.

![Interface web](captures/img4.png)  
*Visualisation en temps réel des visites par page (P1 en vert, P2 en rouge)*

![Logs Kafka Streams](captures/img5.png)  
*Logs montrant les agrégats calculés par Kafka Streams*

---
## Fonctionnalités implémentées
- ✅ **Producer Kafka** via un contrôleur REST (`/publish`)
- ✅ **Consumer Kafka** pour la consommation d’événements
- ✅ **Supplier Kafka** générant des événements simulés automatiquement
- ✅ **Traitement en temps réel** avec **Kafka Streams** (fenêtrage temporel, agrégation)
- ✅ **API interactive** exposant les résultats d’analyse via **Interactive Queries**
- ✅ **Interface web en temps réel** avec graphiques dynamiques (via Server-Sent Events)
---
## Structure du projet
```
EventDrivenArchitecture_KAFKA/
├── src/
│   └── main/
│       ├── java/
│       │   └── b.m29.eda_kafka/
│       │       ├── events/PageEvent.java                  # DTO des événements
│       │       ├── controllers/PageEventController.java   # REST endpoints (producer + analytics)
│       │       └── handlers/PageEventHandler.java         # Consumer, Supplier & Kafka Streams
│       └── resources/
│           └── static/
│               └── analytics.html         # Page web de visualisation 
├── docker-compose.yml                     # Kafka + ZooKeeper via Docker
├── pom.xml                                # Dépendances Maven
└── README.md
```
---
## Architecture Kafka utilisée

| Composant        | Topic / Store             | Description |
|------------------|---------------------------|-------------|
| Producer REST    | `page-events`             | Envoie des `PageEvent` via HTTP |
| Supplier         | `page-event-supplier-out-0` | Génère des événements aléatoires chaque seconde |
| Kafka Streams    | `count-store` (WindowStore) | Compte les événements par page sur une fenêtre de 5s |
| Web Analytics    | SSE (`/analytics`)        | Expose les agrégats via Interactive Queries |

> ✨ Ce projet démontre les capacités de Kafka à gérer des flux d’événements en temps réel, avec une intégration fluide dans une architecture Spring moderne.

---

## Conclusion

Cette activité a permis de :
- Mettre en place un cluster Kafka **en local** et **via Docker**,
- Produire et consommer des événements avec **Spring Cloud Stream**,
- Générer des flux simulés avec un **Supplier**,
- Réaliser un **traitement en streaming** avec **Kafka Streams** (fenêtrage, agrégation),
- Exposer et visualiser les résultats en **temps réel** via une interface web.

L’architecture orientée événements démontre sa puissance pour les systèmes distribués modernes : **découplage**, **scalabilité**, **résilience**, et **réactivité**.