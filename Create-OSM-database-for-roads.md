# Создаем базу данных OSM для дорог, городов и зданий

## Установим все необходимые инструменты и библиотеки

1. Установим PostgreSQL
    ```console
   $ sudo apt install postgresql
    ```
   Проверим, что PostgreSQL установился
   ```Console
   $ psql --version
    ```
   Должно вывести что-то типа: **psql (PostgreSQL) 14.9 (Ubuntu 14.9-0ubuntu0.22.04.1)**
2. Установим расширение postgis 
   ```Console
   $ sudo apt install postgresql-14-postgis-3
   ```
3. Установим go
    ```Console
   $ sudo apt install golang-go
   ```
   Проверим версию go
    ```Console
   $ go version
   ```
   Должно вывести что-то типа **go version go1.18.1 linux/amd64**
4. Установим **imposm3** - утилита для конвертации данных в **postgres** базу данных из сжатого бинарного формата pbf
    ```Console
    $ sudo apt-get install libleveldb-dev
    $ sudo apt-get install libgeos-dev
    $ go install github.com/omniscale/imposm3/cmd/imposm@latest
   ```
   Не забудем добавить в $PATH подгруженную библотеку
   ```Console 
   $ export PATH=$PATH:/home/omela/go/bin
   ```
## Подгрузим необходимые данные
1. Скачаем всю OpenStreetMap карту России с **[geofabrik](http://download.geofabrik.de/russia.html)**
   ```Console
   $ wget -P /home/omela/osm_data -o russia-latest.osm.pbf http://download.geofabrik.de/russia-latest.osm.pbf
   ```
    
   - после <shortcut>-P</shortcut> идет название директории, в которую сохраняем;
   - после <shortcut>-o</shortcut> название, под которым сохраним данный файл
2. Создадим базу данных postgres с расширеним PostGIS
   ```Console
   $ sudo -u postgres -i
   $ createdb osm_data
   echo 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;' | psql -d osm_data
   ```
3. Изменим пароль у пользователя postgres
   ```Console
   omela@omela-virtual-machine:~$ sudo -u postgres -i
   postgres@omela-virtual-machine:~$ psql
   postgres=# ALTER USER postgres WITH PASSWORD 'postgres';
   ```
   
4. Создадим файлик city_mapping.yml и заполним. Его содержимое лежит в /data/city_mapping.yaml

```yaml
tables:
  buildings:
    type: "polygon" # тип геометрии используемый для таблицы, по сути, говорим что будем брать данные из таблички multipolygons
    mapping: # фильтрация по наличия поля, и значению в этом поле
      building: [__any__] # любое не NULL значение, какие значения бывают https://wiki.openstreetmap.org/wiki/RU:Key:building
    columns: # перечисляем поля таблицы
      - {name: osm_id, type: id}                                    # идентификатор OpenStreetMap
      - {name: geometry, type: geometry}                            # геометрия
      - {key: name, name: name, type: string}                       # поле с именем объекта
      - {key: "addr:street", name: street, type: string}            # улица
      - {key: "addr:postcode", name: postcode, type: string}        # почтовый индекс
      - {key: "addr:city", name: city, type: string}                # город
      - {key: "addr:housenumber", name: housenumber, type: string}  # номер дома
      - {key: "addr:quarter", name: quarter, type: string}          # квартал
  cities:
    type: "polygon" # тип геометрии используемый для таблицы
    mapping:
      place: # кто есть кто можно посмотреть тут https://wiki.openstreetmap.org/wiki/RU:Key:place
        - city                 # крупные города
        - town                 # средний или малый город
        - village              # посёлок городского типа
        - hamlet               # Любой сельский населённый пункт размером от двух-трёх домашних хозяйств, не подходящий под критерии village
        - allotments           # всякие  СНТ, ДНТ и т.п.
        - isolated_dwelling    # хутор
        - neighbourhood        # микрорайоны, кварталы и т.п.
        - suburb               # пригороды
        - locality             # заброшенные деревни, урочище
    columns:
      - {name: osm_id, type: id}
      - {name: geometry, type: geometry}
      - {name: type, type: mapping_value}
      - {key: name, name: name, type: string}
      - {key: "addr:country", name: country, type: string}
      - {key: "addr:district", name: district, type: string}
      - {key: "addr:region", name: region, type: string}
      - {key: "addr:postcode", name: postcode, type: string}
      - {key: population, name: population, type: string}
      - {key: "population:date", name: population_date, type: string}
      - {key: official_status, name: official_status, type: string}
```

5. Заполним базу данных для городов и зданий
   ```Console
   $ imposm  import -mapping osm_conf/city_mapping.yaml -read osm_data/data.osm.pbf 
   -write -connection postgis://postgres:postgres@localhost/osm_data

   ```
   Качается примерно за полтора часа
6. Создадим файлик road_mapping.yaml
   **FUTURE**
   ```yaml
   tables:
   ways:
      type: linestring
      mapping:
        highway: [__any__]
   columns:
    - name: osm_id
      type: id
    - name: geometry
      type: geometry
    - name: tags
      type: hstore_tags
     ```

7. Заполним базу данных дорог
     ```Console
      $ imposm  import -mapping osm_conf/road_mapping.yml 
     -read osm_data/russia-latest.osm.pbf -write 
     -connection postgis://postgres:postgres@localhost/osm_data
     ```
   Примерно 1 час 16 минут
  
Так качается долго попробуем сначала только с http://download.geofabrik.de/russia/north-caucasus-fed-district.html

```Console
   $ wget -P /home/omela/osm_data http://download.geofabrik.de/russia/central-fed-district-latest.osm.pbf 
   $ imposm  import -mapping osm_conf/
     -read osm_data/ -write 
     -connection postgis://postgres:postgres@localhost/osm_data
 ```

## Работа с базой данных дорог

По умолчанию координаты хранятся в WSG84 (В этой системе координат 
местоположения на земной пов\ерхности определяются двумя числами: 
широтой (от -90 до 90 градусов) и долготой (от -180 до 180 градусов)), 
но эта система координат не очень 
удобна для поиска кратчайшего расстояния в метрах, поэтому переведем 
координаты в более удобную для этого систему Lambert Conformal Conic (LCC).

```SQL
    SELECT osm_id,        
           ST_Distance(ST_ClosestPoint(r.geometry, point.geom), point.geom)
    FROM import.osm_roads r,       
         (Select ST_Transform(ST_SetSRID(ST_MakePoint(55.810562, 37.933520), 4326), 3857) as geom) point
    ORDER BY r.geometry <-> point.geom ASC LIMIT 1;
```

