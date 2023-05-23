# Based on 

* https://github.com/DigitalPebble/storm-crawler/tree/master/external/opensearch
* https://github.com/DigitalPebble/stormcrawler-docker


# Webseiten crawlen mit StormCrawler 


Sie kennen hoffentlich meine FileMaker Suchmaschine fm-search.com

Dort finden Sie viele Inhalte zu FileMaker und können vor allem schön aufgrund von Autor, Keywords, Betriebssystem filtern und Inhalte rasch finden.

Die Inhalte auf fm-search.com kommen teilweise aus dem gedruckten FileMaker Magazin vom K&K Verlag, wo die PDF Dokumente der verschiedenen FileMaker Magazin Ausgaben per Apache Tika geparst, in JSON umgewandelt und an Elastic Search weitergeleitet werden, damit Sie diese dann auf fm-search.com durchsuchen können.

Der Großteil der Informationen wird direkt online aus den Webseiten ausgelesen, also gecrawlt.

## Crawler4j

Technisch kam hier bis jetzt der OpenSource Crawler crawler4j zum Einsatz. Dieser “besucht” die einzelnen Webseiten, so wie Ihr Webbrowser, liest diese aus, gibt den HTML Quelltext zurück und sucht weitere Links im Dokument, um weitere Seiten zu besuchen.

Den Quelltext speichere ich dann in einer MySQL Datenbank inklusive Informationen, wann die Webseite zuletzt besucht wurde, da es ja nicht notwendig ist, die selbe Webseite immer und immer wieder erneut zu crawlen.

Die meisten Seiten besucht mein Crawler nur alle 14 Tage. Ein paar spezielle Seiten, wie zB die Startseite des FileMaker Magazin Forum wird alle 5 Minuten gecrawlt, damit neue Beiträge im Forum auch sofort in fm-search.com auffindbar sind.

## Extraktionsprozess

Die HTML Seiten werden dann gereinigt (nicht alle Webseiten sind fehlerfrei programmiert 🙂 ) und der innere Aufbau wird analysiert, sodass der Haupttext, Keywords, Autor, Datum etc. extrahiert und in fm-search.com einzeln dargestellt werden kann.

Das sind alles sehr aufwändige, fehleranfällige und ressourcenfressende Prozesse, in die sehr sehr sehr viel Hirnschmalz und Arbeit geflossen ist.

## Datenbankumbau

Nun ist mein eigener, auf crawler4j basierender Crawler aber an seine Grenzen gestoßen und er skaliert nicht mehr so, wie ich mir das vorstelle. Vor allem die Speicherung in MySQL erwies sich nicht als besonders klug, da die Webseiten dort nur als komprimierter Text gespeichert sind und ich zB die extrahierten Links nicht extra speichere und das in einer relationalen Datenbank weniger Sinn machen würde. Dafür eignet sich vermutlich eine Dokumentendatenbank wie MongoDB besser. Oder Elastic Search bzw. dem OpenSource Fork OpenSearch.

Ich habe dann angefangen meinen Crawler auf MongoDB umzubauen, dachte mir dann aber, ich schau einmal, was es online noch so an Crawler gibt, da die Umbauarbeiten doch ziemlich intensiv anmuteten. 

Bei solchen Projekten lerne ich immer extrem viel und kann das ganze meistens später bei Kundenprojekten auch verwenden, aber da fm-search.com ein kostenloses Angebot ist, für das ich noch nie auch nur 1 Cent gesehen habe, war mir das dann doch irgendwie zu viel Arbeit für ein Hobbyprojekt.

## StormCrawler

Aus diesem Grund bin ich dann über StormCrawler gestolpert.

StormCrawler ist ein – genau – Crawler, der auf Apache Storm basiert. Von Storm hatte ich noch nie gehört und wollte damit auch nicht all zu viel zu tun haben, da das ein System für das Verteilen von Jobs auf vielen Nodes ist, was ich für fm-search.com nicht brauche. Die paar FileMaker Seiten kann ein einzelner Linux Server locker alleine crawlen.

Wenn fm-search.com mal die Größe von Google erreicht 🙂 dann braucht es noch ein paar Nodes mehr, aber bis dahin ist noch ein weiter Weg. 

Ich habe dann erst mal aufgrund dieses Blogbeitrages versucht StormCrawler ohne Storm zum Laufen zu bekommen, aber das klappte nicht. ElasticSearch hatte ich nämlich in einem Docker Container am Laufen und das klappte so nicht. Mit den Details will ich Sie jetzt nicht langweilen.

Ich habe dann Julien Nioche, dem Hauptentwickler hinter StormCrawler, kontaktiert und wollte von ihm Support einkaufen, um StormCrawler up-and-running zu bekommen.

Julien war so freundlich mir per Teamviewer zu helfen, das Projekt zu starten. Er wollte kein Geld von mir nehmen und wir haben uns dann darauf geeinigt, dass ich dem EB Haus eine Spende zukommen lasse, was ich natürlich gerne gemacht habe.

Ich habe außerdem angeboten, meine Eindrücke und First-Steps zu und mit StormCrawler zu verfassen, was ich hiermit gerne öffentlich mache.

## StromCrawler mit Docker und OpenSearch Tutorial

Also, los gehts:

~~~
#Verzeichnis anlegen und reinwechseln
mkdir stromcrawler-tutorial && cd "$_" 

#Java 11 in Shell aktivieren. Ich verwende dazu skdman
sdk use java 11.0.13-librca            

#Java Projekt anlegen mit maven
mvn archetype:generate -B                                  \
  -DarchetypeGroupId=com.digitalpebble.stormcrawler        \
  -DarchetypeArtifactId=storm-crawler-opensearch-archetype \
  -DarchetypeVersion=2.8                                   \
  -DgroupId=com.schubec                                    \
  -Dversion=1.0-SNAPSHOT                                   \
  -DartifactId=fmcrawler

#Ins Projektverzeichnis wechseln
cd fmcrawler
~~~

In der Fernwartungssession mit Julien habe ich gelernt, dass man StormCrawler auf zwei verschiedene Arten konfigurieren kann – einmal im Java Quellcode, was mir persönlich eher liegt oder indem man sogenannte Flux-Dateien konfiguriert. 

Die Flux-dateien sind gewöhnliche Textdokumente, die beschreiben, wie Webseiten gecrawlt und geparst werden. Diese haben natürlich den Vorteil, dass das Projekt konfiguriert werden kann, ohne dass man jedesmal alles mit Java kompilieren muss.

Als nächstes Kompilieren wir jetzt einmalig den generierten Quellcode (später Anpassungen dann nur noch in den Flux-Dateien) und geben dem Crawler eine Startadresse.

~~~
# Alles durchkompilieren
mvn package

#Start-URL in seeds.txt Datei schreiben
echo "https://www.schubec.com" > seeds.txt 
~~~

## OpenSearch und weitere Komponenten in Docker

Als nächstes müssen wir OpenSearch, den Fork von ElasticSearch, installieren. 

Da ich meinen Rechner nicht mit unnötig Software zumüllen möchte, werden OpenSearch plus zugehörige Komponenten in Docker Container installiert.

Hierzu gibt es eine Vorlage, die ein Jahr alt ist und ein wenig aktualisiert und angepasst wird. Mein docker-compose.yml File sieht daher so aus:

~~~
version: '2.4'

services:
  zookeeper:
    image: zookeeper:3.7.0
    container_name: zookeeper
    restart: always

  frontier:
    image: crawlercommons/url-frontier
    container_name: frontier
    command: rocksdb.path=/crawldir/rocksdb
    ports:
      - "127.0.0.1:7071:7071"
    volumes:
      - ./data/stormcrawler-docker:/crawldir

  builder:
    image: schubec/storm_maven:2.4.0-temurin
    container_name: builder
    depends_on:
      - nimbus
    volumes:
      - ./data/crawldata:/crawldata

  nimbus:
    image: storm:2.4.0-temurin
    container_name: nimbus
    command: storm nimbus
    depends_on:
      - zookeeper
    restart: always
    volumes:
      - ./data/storm-nimbus-logs:/logs

  supervisor:
    image: storm:2.4.0-temurin
    container_name: supervisor
    command: storm supervisor -c worker.childopts=-Xmx%HEAP-MEM%m
    depends_on:
      - nimbus
      - zookeeper
    restart: always
    volumes:
      - ./data/storm-supervisor-logs:/logs

  ui:
    image: storm:2.4.0-temurin
    container_name: ui
    command: storm ui
    depends_on:
      - nimbus
    restart: always
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - ./data/storm-ui-logs:/logs

  elasticsearch:
    image: elasticsearch:7.16.3
    container_name: elasticsearch
    restart: always
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms750m -Xmx3750m"
    volumes:
      - ./data/elasticsearch/data:/usr/share/elasticsearch/data
    logging:
      driver: "json-file"
      options:
        max-file: "3"
        max-size: "100m"
    mem_limit: 4096m
    ports:
      - 9200:9200
     
  kibana:
    image: kibana:7.16.3
    container_name: kibana
    restart: always
    environment:
      OPENSEARCH_HOSTS: http://elasticsearch:9200
    depends_on:
      - elasticsearch
    logging:
      driver: "json-file"
      options:
        max-file: "3"
        max-size: "100m"
    mem_limit: 1024m
    ports:
      - 5601:5601
~~~
      
Das Image `schubec/storm_maven:2.4.0-temurin` muss man selbst bauen, was mit folgendem Befehl `docker build -t schubec/storm_maven:2.4.0-temurin .` und diesem Dockerfile leicht geht:

~~~
FROM storm:2.4.0-temurin

# Install maven
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive \
    apt-get install -y maven software-properties-common openjdk-11-jdk\
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*  

ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64

USER storm
~~~

Das gesamte System kann nun wie gewohnt mit `docker-compose up -d` gestartet werden.