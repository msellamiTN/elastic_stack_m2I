# Atelier 5 : Déployer Elastic Stack avec Docker Compose
## Étape 1 : Cloner le Référentiel
Commencez par cloner le référentiel contenant les configurations nécessaires.
```bash
git clone https://github.com/msellamiTN/elastic_stack_cyber.git
cd elastic_stack_cyber
cd docker_elk
```
## Étape 2 : Lancer le Cluster Elastic Stack

```bash
docker-compose up -d
```
## Étape 3 : Copier les Certificats depuis le Master es01
Créez un dossier pour les clés et copiez le certificat CA depuis le conteneur Elasticsearch.
```bash
mkdir keys
sudo docker cp elastic_stack-es01-1:/usr/share/elasticsearch/config/certs/ca/ca.crt ./keys/ca.crt
sudo chmod 644 ./keys/ca.crt
```
## Étape 4 : Démarrage des Services avec Docker Compose
Lancez les services définis dans le fichier docker-compose.yml.
```bash
docker-compose up
```
## Étape 2 : Télécharger l'image Docker de logstash
Pour télécharger l'image Docker de logstash à partir du registre Docker Hub, vous pouvez utiliser la commande suivante dans votre terminal :
```bash
 docker pull docker.elastic.co/logstash/logstash:8.13.4
```
Assurez-vous que le conteneur Elasticsearch est déjà en cours d'exécution avant de démarrer logstash.

## Étape 3: Préparation de pipepline Logstash :
Placez vos fichiers JSON d'emails dans un répertoire spécifique sur votre système. Assurez-vous que les fichiers suivent la structure JSON fournie.

- Création du fichier de configuration Logstash  logstash.yml  et copier de dans le contenu suivant :
```bash
mkdir logstash
cd logstash
mkdir -p logstash/config
nano  logstash.yml 
```
copier ce contenu dans le fichier logstash.yml 
```yml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "https://es01:9200"]
```
- Créez un pipeline de configuration nommé logstash.conf. 
les commandes shell à suivre pour créer un pipeline de configuration
```bash
mkdir -p logstash/pipeline
nano  logstash.conf 
```
copier le contenu suivant dans ce fichier logstash.conf :
```conf
input {
  file {
    path => "/data/files/*.json"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    codec => "json"
  }
}

filter {
  json {
    source => "message"
    skip_on_invalid_json => true
  }

  if [type] == "email" {
    # Parse JSON fields and extract relevant information
    json {
      source => "message"
      target => "email_data"
    }
    
    # Filter out events without attachments
    if ![email_data][attachments] {
      drop {}
    }
    
    mutate {
      add_field => {
        "sender" => "%{[email_data][sender]}"
        "recipient" => "%{[email_data][recipient]}"
        "date" => "%{[email_data][date]}"
        "subject" => "%{[email_data][subject]}"
        "body" => "%{[email_data][body]}"
        "tags" => "%{[email_data][tags]}"
        "attack_type" => "%{[email_data][attack_type]}"
      }
    }
    
    # Extract attachment details
    ruby {
      code => '
        attachments = event.get("[email_data][attachments]")
        if attachments
          event.set("attachments", attachments.collect { |attachment| attachment["filename"] })
          event.set("attachments_content_type", attachments.collect { |attachment| attachment["content_type"] })
        end
      '
    }
    
    # Extract IOC details
    ruby {
      code => '
        iocs = event.get("[email_data][ioc]")
        if iocs
          event.set("ioc_indicator", iocs.collect { |ioc| ioc["indicator"] })
          event.set("ioc_type", iocs.collect { |ioc| ioc["type"] })
        end
      '
    }
    
    # Remove temporary fields
    mutate {
      remove_field => ["message", "email_data", "[email_data][attachments]", "[email_data][ioc]"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["https://es01:9200"]
    index => "logstash_email_attacks-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "1Cge99g6yEs3s4406vWk"
    ssl => true
    cacert => "/usr/share/logstash/config/certs/ca/ca.crt"
  }
  
  stdout {
    codec => rubydebug
  }
}
```
## Étape 4 : Configurer et demarrer le service Logstash
Une fois que la configuration est terminée, definir le service logstash en ajoutant ce script dans le fichier docker-compose.yml en editant le fichier avec nano ou un autre editeur
### Configurer le service logstash
```yml
 logstash:
    image: docker.elastic.co/logstash/logstash:${STACK_VERSION}
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
      - ./logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./logstash/data/files:/data/files
      - ./keys/ca.crt:/usr/share/logstash/config/certs/ca/ca.crt
    environment:
      - ELASTICSEARCH_HOST=https://es01:9200
      - ELASTICSEARCH_USER=${ELASTICSEARCH_USERNAME}
      - ELASTICSEARCH_PASSWORD=${ELASTIC_PASSWORD}
    ports:
      - "5044:5044"
```
### Exécutez la commande docker compose suivante pour démarrer Logstash.
```sh
docker compose up -d logstash
```
