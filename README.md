# Filebeat and ELK

Demo jest uruchamiane za pomocą `docker-compose`.
Prezentuje w jaki sposób za pomoca narzędzia Filebeat można automatycznie aggregować logi w ELK.
Filebeat monitoruje pliki z logami `/var/lib/docker/containers/*/*log` i wysyła je do ElasticSearch

## Środowisko testowe
Wymagania:
* Debian 10
* docker-ce
* docker-compose

## Instalacja docker-ce
```
sudo apt update
sudo curl -fsSL get.docker.com | sh
```

## Instalalcja docker-compose
```
# https://docs.docker.com/compose/install/
sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## Zmiana ustawień kernal
```
sysctl -w vm.max_map_count=262144
```

## Uruchomienie kontenerów
```
docker-compose up -d
```

`docker-compose` tworzy kontenery:
* Kibana dostępna na porcie 5601/TCP
* ElasticSearch dostępny na porcie 9200/TCP
* Filebeat - plik configuracyjny w `filebeat.docker.yml`
* web - kontener testowy z Apache2
* nginx - kontener testowy z Nginx


## Konfiguracja Kibana
1. Przejdź do Menu -> Management -> Stack Management
2. Następnie po lewej stronie wybierz Index Patterns z Kibana.
3. Naciśnij 'Create index pattern'.
4. W 'Index pattern name` wpisz 'filebeat-*' i naciśnij `Next`.
5. Z menu rozwijanego wybierz `@timestamp` i naciśnij 'Create index pattern'
6. Przejdź do głównego menu i wybierz 'Kibana' -> 'Discover' aby zobaczyć logi wysłane przez `Filebeat`.


## Format logów - JSON
Test wysyłania logów w formacie JSON.
Polega na uruchomieniu kontenera z wypisaniu na stdout logu w formacie JSON.
1. Imitate json log
```
$ docker run  --rm -l json_logger=True --name my_app  ubuntu echo '{ "host": "192.168.0.1", "message": "test22"} '
```

2. Konfiguracja Filebeat procesuje wg swojej konfiguracji linie w formacie JSON wysyłane z konterów oznaczonych przez 'label'  jako 'json_logger=True'. Logi są 'rozbijane` przez Filebeat na poszczególne pola np. 'json.host' i 'json.message'.
```
processors:
- decode_json_fields:
    when.or:
      - contains:
          docker.container.labels.json_logger: "True"
      - contains:
          docker.container.labels.json_logger: "true"
    fields: ["message"]
    target: "json"
    overwrite_keys: true
```
3. Filebeat po przeprocesowaniu wysyła do ES log podony do poniższego.
```
{
  "_index": "filebeat-7.9.2-2020.10.09-000001",
    "json": {
      "host": "192.168.0.1",
      "message": "test223"
    }
  }
}
```
4. Po przejściu do Kibana -> Discovery, dodajemy nowy filter 'Add filter' i wpisujemy:
Field: container.name
Operator: is
Value: 'my_app'

Po zapisaniu zostaną wyświetlone logi tylko z kontenera 'my_app'.

5. Kontener flaskapp w `docker-compose.yml` uruchamia mikroserwis napiany we Flask i generuje logi w formacie JSON.
Logi można wygenerować wysyłając zapytania
```
$ curl -I localhost:5000
$ curl -I localhost:5000
$ curl -I localhost:5000
```

6. Dla porównania logi z Kibana dla flaskapp można porównać z logami z kontenera wydając polecienie
```
docker-compose logs -f flaskapp
```

## Logi Apache2 i Nginx
Filebeat posiada w budowaną funkcjonalność do procesowania logów z Apache2 i Nginx.
Według konfiguracji Filebeat:
* logi z Apache2 będą procesowane jeżeli zostanie dodany label 'docker.container.labels.apache2=True' do kontenera.
* logi z Nginx bedą procesowane jeżeli zostanie dodany label 'docker.container.labels.nginx' do kontenera.

Poniższe komendy wygenerują logi na serwer apache2 i nginx.
```
$ curl -I localhost:80
$ curl -I localhost:80
$ curl -I localhost:80
$ curl -I localhost:81
$ curl -I localhost:81
$ curl -I localhost:81
```

W Kibana jest możliwość filtrowania logów z konterów web i nginx przez modyfikacje filtru dla pola 'container.name'.
