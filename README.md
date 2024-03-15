# laravel-elastic-models

Teljeskörű elastic integráció Laravel 8+ -hoz.

- Függőség: `cviebrock/laravel-elasticsearch` (composer be fogja húzni ezt is, konfigurálnunk kell csak lsd. később)

## Telepítés

Aztán a laravel rootban:

```
composer require ...../laravel-elastic-models
```

Mivel a package composer.json -jában szerepel a

```
"extra": {
    "laravel": {
      "providers": [
        "......\\LaravelElasticModels\\Providers\\LaravelElasticModelsServiceProvider"
      ]
    }
  }
```

rész, így a laravel meg fogja találni megadott provider-t, tehát regisztrálódnak az ott megadott dolgok
(migration fileok, eventek, stb).

És végül ezzel tudjuk alkalmazni, mint laravel package (alkalmazza a providert és tartalmát):

```
php artisan vendor:publish
 Which provider or tag's files would you like to publish?:
  [0 ] Publish files from all providers and tags listed below
  [1 ] Provider: Cviebrock\LaravelElasticsearch\ServiceProvider
  [2 ] Provider: .........\LaravelElasticModels\Providers\LaravelElasticModelsServiceProvider
```

itt (ha még nem volt előzően) a `Cviebrock\LaravelElasticsearch\ServiceProvider`-t majd
a `........\LaravelElasticModels\Providers\LaravelElasticModelsServiceProvider` publikáljuk.

Ezután a `config/elasticsearch.php` -ben ezeket a paramétereket így kell beállítan, ha az SSL miatt nem sikerül később
csatlakozni:

```
 'sslVerification' => false,
 'sniffOnStart' => true,
```

Utána:

```
php artisan migrate
```

hogy az elastic időzített szinkronációhoz meglegyen a tábla `elastic_synsc`.

A `.env` -ben pedig a állítsuk be az elastic serverhez az adatokat:

```
ELASTICSEARCH_HOST=ites1........com
ELASTICSEARCH_HOST2=ites2.......com
ELASTICSEARCH_HOST3=ites3......com
ELASTICSEARCH_PORT=9200
ELASTICSEARCH_SCHEME=https

ELASTICSEARCH_USER=adverts_test
ELASTICSEARCH_PASS=iem5aeN4mu8p
ELASTICSEARCH_INDEX_PREFIX=test_
ELASTICSEARCH_INDEX_SUFFIX=_test

```

## Service struktúra

A provider definiál egy `Elastic::` nevű `Facade`-t amivel két service-t tudunk meghívni:

```
Elastic::index(); //.....\LaravelElasticModels\Services\ElasticSearchService indexszeléssel kapcsolatos műveletek
Elastic::index()->updateByQuery($model, [...]) //pl

Elastic::query(Advert::class); //.......\LaravelElasticModels\Services\FilterService ( querybuilder, lásd később)
```

## Elastic indexek tartalmainak megtekintése

`APP_DEBUG=true` -nál tudjuk az  
`/elasticrecords/{limit?}/{ModelName?}`
végponton megnézni a elastic indexek rekordjait (= ElasticSearchModelInterface megvalósító eloquent modellekhez tartozó
db táblák elastic index megfelelői)
(ha nem adunk meg limitet, akkor 100 a limit az összesnél, és 1000 ha megadjuk a `ModelName`-et)

A rendezési infókat GET paraméterekben lehet megadni: `?sort_field=id` és `&sort_dir=desc`

## Indexeléssel kapcsolatos feature-ök

### Model-ek inicializálása

- `php artisan elastic:reindex`
  - Csak azt a laravel modelt dolgozzza fel (automatikusan), amelyik implementálja az  `ElasticSearchModelInterface`
    -t.
  - Ha egy táblához több Model tartozik, akkor a felhúzás/újrahúzás csak egy modelnél lehet eszközölni, ilyenkor a többi
    Model-nél `not_reindexable` publikus mezőt kell definiálni true érétékkel
  - Az indexek nevei a modelekhez tartozó db tábla nevei lesznek, .env-ben adhatunk meg prefixet, ha a teszt ugyanott
    van mint az éles: `ELASTICSEARCH_INDEX_PREFIX=test_`
  - A `getElasticSearchColumnsAndTypes()`-ban kell megfeleltetni az egyes attribútumokat az elastic típusokkal.

```
 return [
            'id' => ['type' => 'keyword'],
            'email_body' => ['type' => 'text', 'analyzer' => 'ignore_accents_and_case_but_save_original_accents_too'],
            'email' => ['type' => 'keyword'],
        ]
```

    - Tudunk dinamikusan is létrehozni elastic mezőt egylőre csak sum-mal:

```
public function getElasticSearchGeneratedColumns(): ?array
{
  return [
    'ad_room_all' => ['op' => 'sum', 'fields' => ['ad_room', 'ad_room_half']]
  ];
}
```

#### Mező beállítása fulltext kereséshez

Ha egy text mezőt fulltext módon kell kereshetővé tennünk, akkor a benne lévő szöveget "tokenizálnunk" kell, tehát a
szöveg szavait egyenként indexelni, hogy ha fulltext keresés van, akkor a keresett szavakra külön külön keressen rá, és
amelyik találatban több keresett szó fordul elő, az lesz a relevánsabb rekord, azt fogja előre sorolni a találatokban.
Ehhet `analyzer`-re van szükségünk, amiből jelenleg 3db van definiálva:

```
  'email_body' => ['type' => 'text', 'analyzer' => 'ignore_case'],
  'email_body' => ['type' => 'text', 'analyzer' => 'ignore_accents_and_case'],
  'email_body' => ['type' => 'text', 'analyzer' => 'ignore_accents_and_case_but_save_original_accents_too'],

```

- `ignore_case` -nél lowercase-ben indexeli a mezőt (de ékezettel a szavakat)
- `ignore_accents_and_case` -nél lowercase-ben, és ékezet nélkül
- `ignore_accents_and_case_but_save_original_accents_too` -nél lowercase-ben, és ékezet nélkül + elmenti ékezettel is a
  szavakat (lsd. 3. paraméter: whereFulltext(['email_body',$searchText, true]))

### Újrahúzás nélküli csoportos reindexelés

Ezeket parancsokat csak akkor lehet használni az adott laravel modelnél, ha a modelnek van egy UTC időben tárolt "utolsó
módosítási idő" attribútuma és ha ez eltér az `updated_at` -től akkor publikus property-ként hozzá van
adva `public $updatedAtUTCColumn` néven az eloquent modelhez.

- `php artisan elastic:ontheflyreindex {all?} `: (cronjobhoz) az utolsó ilyen reindex időpontja óta ha van módosított
  rekord, azt automatikusan szinkronizálja
- `php artisan elastic:ontheflydelete`: (cronjobhoz) az utolsó ilyen reindex időpontja óta ha van törölt elem, törli az
  elasticból is

## Laravel Model CRUD műveletek

Amelyik laravel model implementálja a `ElasticSearchModelInterface`-t annak a rekordjait az egyes (Eloquent) CRUD
műveleteknél szinkronban tartja az elasticcal.  
`Updated` eseménynél van lehetőségünk megakadályozni a szinkront:
Ha létrehozunk egy `public $dontSaveToElastic` property-t az adott eloquent modelben, és a futás során `true` -ra állítjuk.
Ezt jellemzően akkor használjuk, ha olyan mezőt frissítettünk, amit nem vettünk fel elasticban, így fölöslegesen küldenénk be az egész modelt.

### Update where... (update_by_query)

Tudunk elastic nodeokat feltétel alapján is updatelni (nem csak eloquent model-eket egyenként vagy iterálva), ezt a

 ```
 Elastic::index()->updateByQuery($model, $elasticBody)
 ```

biztosítja. Itt is meg kell adnunk az érintett eloquent model egy példányát, de ez a példány lehet akár üres eloquent
model is, mert a második paraméternek megadott elastic specifikus php tömbnek kell tartalmazia az érdemi adatokat (=
elastic api üzenet body része). többit lsd elastic update_by_query.

## QueryBuilder

### WHERE feltételek megadása

#### 1. verzió: függvényekkel rakjuk össze

Az `Elastic::query(Advert::class)` vel tudunk elastichoz query buildert használni itt az `FilterServiceInterface` -ben
,megadott függvények egyértelműek (onlyCount(), limit() stb).

```
->find($id, $asArray = false)

->where($attr, $val);
->where($attr,'=', $val);
->where($attr,'!=', $val);
->where($attr,'<', $val);
->where($attr,'<=', $val);
->where($attr,'>', $val);
->where($attr,'>=', $val);
->whereIn($attr, array $vals);
->whereNotIn($attr, array $vals);
->whereIds($attr, $ids);

```

**Fulltext keresés, kulcsszó keresés**

**fulltext** (több szavas kifejezés) keresés és a **kulcsszó** (egyszavas) keresés találatai egyaránt relevancia (`_score`) szerint lesznek rendezve, 
így hasonlóan kell ezeket intézni: 

```
->whereFullText($attrOrAttrsWithWeight, $expression, $withAccents = false, $fulltextType = self::FULLTEXT_MOST_FIELDS, $matchPercent = '100%'): FilterServiceInterface;
```

- `$attrOrAttrsWithWeight` lehet string `email_body`, vagy array `['email_body' => 20, 'email_subject' => 30]` a súly értelemszerűen minél nagyobb, annál relevánsabb a mező
- Ha `$withAccents = true`, akkor az adott mezőnél a Model-ben be olyan analyzert kell beállítani, ami menti ékezettel
  is a szavakat
- `$fulltextType` FilterServiceInterface::FULLTEXT_PHRASE => "kifejezés" (így ez nem is fulltext), FilterServiceInterface::FULLTEXT_MOST_FIELDS => klasszikus fulltext, (kulcsszónál nem használható)
- `$matchPercent` a kifejezés szavainak minimum hány %-a legyen benne


```
'email_body' => ['type' => 'text', 'analyzer' => 'ignore_case'],
'email_body' => ['type' => 'text', 'analyzer' => 'ignore_accents_and_case_but_save_original_accents_too'],
```

ha az analyzer-nek `ignore_accents_and_case` lett beállítva, akkor nem fogja ékezettel megtalálni a szavakat.

Query builderrel csak EGY DARAB fullText blokk lehetséges, ha bonyolítani szeretnénk, akkor rawQuery-t kell írnunk: pl.
``` 
{
  "query": {
    "bool": {
      "should": [
        {
          "multi_match": {
            "query": "SEARCH TERM HERE",
            "fields": [
              "title^70",
              "description^30",
              "content^20"
            ],
            "type": "phrase", //kulcsszó keresés (egy szó) nem használható, csak több szavas keresésnél
            "boost": 100
          }
        },
        {
          "multi_match": {
            "query": "SEARCH TERM HERE",
            "fields": [
              "title^30",
              "description^25",
              "content^10"
            ],
            "type": "most_fields",
            "minimum_should_match": "100%",
            "boost": 50
          }
        },
        {
          "multi_match": {
            "query": "SEARCH TERM HERE",
            "fields": [
              "title^25",
              "description^15",
              "content^10"
            ],
            "type": "most_fields",
            "minimum_should_match": "50%",
            "boost": 25
          }
        },
        stb feltétel...
      ]
    }
  }
}
First multi_match is matching documents only when full search phrase is found and boost entire result with 100.
Second part searches for 100% words from search term. So the order of words does not matter but all of them must appear in searched document. Boost = 50.
Third part searches for 50% match. Means not all words have to be in document to return it in results. Boost = 25.
The ... part means that I have something more for rest of the results. But it is not needed in each case.
The boost values are selected by myself in many tries and could not be good for every single case. You have to remember that behind of relevancy there is a quite complex algorithm. For more info take a look into:
```

gyakorlatilag rengeteg találat megfelelhet csak eltérő score-t fognak kapni, ha azt akarjuk, hogy a relevánsakat küldje csak, akkor:
```
->setMinScore($minimumScore)
```
Lapozásnál (lsd később) meg kell adni explicit sort adatokat, benne legalább egy létező mezővel, ilyenkor így oldjuk meg,
ha relevancia szerint rendezzük:
```
$this->elasticQueryBuilder->addOrderBy('_score', 'desc');
$this->elasticQueryBuilder->addOrderBy('id');
```

(ilyenkor az elemeket nem a `$this->elasticQueryBuilder->get()` -tel iteráljuk, hanem a
hanem a `$this->elasticQueryBuilder->query()->getResponse()` -zal, hogy hozzáférjünk a találatok `_score`-jához.)

**track_total_hits**
Ide tartozik még a `->trackTotalHits($bool)` is:
https://stackoverflow.com/questions/66414349/how-does-elasticsearch-7-track-total-hits-improve-query-speed
A. When index sorting is configured, the documents are already stored in sorted order within the index segment files. So whenever a query specifies the same sort as the one in which the index was pre-sorted, then only the top N documents of each segment files need to be visited and returned. So in this case, if you are only interested in the top N results and you don't care about the total number of hits, you can simply set track_total_hits to false. That's a big optimization since there's no need to visit all the documents of the index.
B. When querying in the filter context (i.e. bool/filter) because no scores will be calculated. The index is simply checked for documents that match a yes/no question and that process is usually very fast. Since there is no scoring, only the top N matching documents are returned per shard.
If track_total_hits is set to false (because you don't care about the exact number of matching docs), then there's no need to count the docs at all, hence no need to visit all documents.
If track_total_hits is set to N (because you only care to know whether there are at least N matching documents), then the counting will stop after N documents per shard.

**OR**

```
->whereOr([
    ['cookie_id', $cookieId],
    ['user_id', '=', $userIds] 
])
VAGY 
->whereOr(['cookie_id', $cookieId]);
->whereOr(['user_id', $userId]);
```



#### 2. verzió: tömb elemekkel rakjuk össze

A mergeWhere() -rel lehet hozzáadni feltételeket, ilyen struktúrában (a fenti fg-ek is ezt használják)

```
->mergeWhere([OPERÁTOR => ['mező neve' => $value]]);
```

Összes lehetőség:

```
  $query = Elastic::query(Advert::class) 
  $query->mergeWhere([FilterServiceInterface::IN => [$column => ['dsds','fdfdfd']]]); 
  $query->mergeWhere([FilterServiceInterface::NOT_IN => [$column => ['dsds','fdfdfd']]]); 
  
  $fromTo= [];
  $fromTo[FilterServiceInterface::GREATER_EQUAL_THAN] = 10;
  $fromTo[FilterServiceInterface::LESS_EQUAL_THAN] = 20;
  $query->mergeWhere([FilterServiceInterface::BETWEEN => [$column => $fromTo]]); 
  
  $query->mergeWhere([FilterServiceInterface::EQUAL => [$column => 'sdsdsdsd']]); 
  $query->mergeWhere([FilterServiceInterface::NOT_EQUAL => [$column => 'sdsdsdsd']]); 
  
  $query->mergeWhere([FilterServiceInterface::IDS => ['ad_id' => [1,2,3,4,5]]]); 
  $query->mergeWhere([FilterServiceInterface::IDS => [1,2,3,4,5]]); 
  
  $query->mergeWhere([FilterServiceInterface::FULL_TEXT => ['email_body' => $expression ]])
  $query->mergeWhere([
                FilterServiceInterface::FULL_TEXT_MULTI_FIELD => ['multi_field' => [
                    'attrs' => ['email_body', 'email_subject'],
                    'value' => $expression]
                ]]);  
  
  ```

### Order by

```
$query->orderBy('email_body');
$query->orderBy('email_body','desc');

$query->addOrderBy('email_subject');
$query->addOrderBy('email_subject','desc');

```

### Eredmények lekérdezése

Ha megadtunk minden feltételt, akkor így tudjuk lekérni az adatokat:

```
    $query->first($asArray = false);
    $query->get($asArray = false); 
    $query->count(); //int
    $query->getElasticResult(); //tömb elastictól kapott struktúrában a találatok, ez pár extra adatot tartalmaz
    
```    

paraméterben true-t adunk meg, akkor a tömbben a találatok tömbök lesznek egyébként meg objektum, igaz, hogy nem
eloquent model, de adatolvasás szempontjéból **ugyanolyan mintha eloquent modelleket iterálnánk $object->field_name**

```
    $query->get(); //a rekordok objektunok lesznek
    $query->get(true); //a rekordok tömbbök lesznek
```

Ha egy lekérdezéssel kapcsolatban több infora van szükségünk, count result, pagination data, akkor a `->query()` fg-nyel
tudunk a lekérdezés után a DataQueryServiceInterface objektumban hozzáférni ezekhez

```
    
    //DataQueryServiceInterface objektumot kapjuk vissza, ahol tudunk debuggolni, + itt is megvannak a  get(), count(), getElasticResult() fg-ek + a pagerData():
    $result = $query->query(); 
    $result = $query->query(true); //debug mode: így megkapjuk az elasticfelé elküldendő összerakott api üzenetet
    
    $result->get(true); //tömbben a találatok[['id'=> 1, 'name' => 'sdsdsds'], ['id'=> 2,'name'=> 'sdsdsdsds']]
    $result->get(); ///tömbben a találatok objektumok lesznek lsd fentebb
    $result->count();
    $result->getElasticResult(); //elastictól kapott struktúrában a találatok, ez pár extra adatot tartalmaz
    $result->getPagerData() //ha pagerrel történő lapozásos lekérdezés van (tehtá sok az adat, itt lehet lkérdezni 
    az aktuális pager adatokat, amit a következő oldal lekéréséhez be kell küldeni, és tudni fogja, mettől meddig küldje 
    a találatokat lsd később Pagination 
```

### Raw query

Ha speciálisabb lekérdezésre van szükségünk, ahol a fenti a `where params` lehetőség nem elég, lehetőségünk van direktbe
megadni a `body.query` részét az elastic lekérdezős jsonnak (php array formában).
Meg is adhatjuk az adott query builder objektumnak, hogy csak raw query-t fogadhat be `->onlyRawQuery()` , ilyenkor 
where...() fg-ek hívásánál exceptiont dob.
lsd: https://www.elastic.co/guide/en/elasticsearch/reference/current/search-your-data.html

```
  $query = Elastic::query(Advert::class) 

    $query->setRawQuery([
         "match" => [
              "user_id"=> 12
        ]
    ]);
    
    $result = $query->get();
```

Lehetőségünk van azz elastichoz menő requestbe mergelni elemeket a `$query->mergeToRequestArray(array $datas)` 
ennek segítségével a request body tehát az érdemleges részéhez tudunk hozzáadni elemeket,pl `search_after`.
ezt nyilván leginkább a bonyolultabb rawQuery-s megoldásoknál lesz aktuális.  

Vissza irányból meg a lekérdezés ($query->get() / count() / query()) után a `$query->getResponseWithoutHits()` fg-nyel tudjuk 
a nyers response üzenetet megkapni (találatok nélkül).

### Pagination

A kvázi limit/offset-es (from/size) megoldás az elasticban maximum 10.000 találatig lehetséges, de néhány ezredik
találatig  
érdemes csak hagyni ezt. Utána speciálisabb eszközökhöz kell nyúlni: több is van:

#### stateless search_after
Ennél ezek a teendők vannak:
- a lekérdezés(ek)ben nem adunk meg `page` -t csak `limit`- et,
- mindenképp kell megadnunk `sort` (orderBy), legalább egy olyan mezővel, ami létező mezőnév pl `id`
- ebben a példában kulcsszavas keresést intézünk, és relevancia (_score) alapján kérjük le az oldalakat
- Az első lekérdezésben  értelem szerűen nincsen `search-after` mező, a 2. oldaltól már el kell küldenünk, úgy, hogy az előző lekérdezés eredményének az utolsó
találatának a `sort` ban meghatározott mezőit elküldjük a `search_after` mezőben megfelelő sorrendben pl. megoldás:

```
        
        (...)
        $lastElement = false;
        
        do {
            if ($lastElement) {
                $this->elasticQueryBuilder->searchAfter([
                        number_format(_score, 12, '.', ''),
                        $lastElement['_id']
                    ]
                );
            }

            $hitsQuery = $this->elasticQueryBuilder->query();
            $hits = $this->getResponse()['hits']['hits'];
            
            foreach ($hits as $hit) {
                $hit = $hit['_source'];
                $hit['mezőneve']...
                (...)
            }

            $lastElement = last($hits);

        } while ($lastElement);

```
Az elküldött ID utáni találatokat fogja küldeni, mivel **stateless**, a keresés alatt történő tartalmi változások nem függetlenek a találatoi listától:
tehát, ha az első oldalon látunk egy elemet, és lapozás közben úgy módosul, hogy a 10. oldalra kerül, akkor a 10. oldalon is fogunk vele találkozni.

(Megj: a `_score` relevancia rendezés esetében nem a `$this->elasticQueryBuilder->get()` -tel kérdezzük le a találatokat, 
hanem a `$this->elasticQueryBuilder->query()->getResponse()` -zal, hogy hozzáférjünk a `_score` adatokhoz, ne csak a mező értékekez lsd fentebb is.)


#### stateful search_after
Point In Time (PIT) azonosítóval ellátott search keresés
Mivel statefull, így a keresés kezdeti állapotot megtartja (X ideig), akkor is,  ha változnak közben az elemek.

https://www.elastic.co/guide/en/elasticsearch/reference/7.x/paginate-search-results.html#search-after

Röviden: Kell készíteni egy  (PIT point in time) azonosítót, amit search api híváshoz csatoljuk, és ezáltal minden
találathoz tartozni fog egy őt azonosító
"sort-tömb" , ami a rendezési adatokat tartalmazza, és amire hivatkozni kell a search API hvásnál a search_after
mezőben. Tehát itt nem azt adjuk meg, hogy melyik oldalt adja vissza, hanem azt, hogy melyik "sort tömb" azonosító utáni
találatokat adja vissza nekünk.

"search_after": [                                
"2021-05-20T05:30:04.832Z", 4294967298
],

Tegyük fel, hogy egy oldal mérete 20 elem, és 3000 találat fölött használjuk a pagination megoldást.

Mivel pár ezer találatig jó a hagyomás limit/offset (from/size) megoldás, így ez megmaradt 3000 találatig (3000/20 =
150.oldalig).  ```config('laravelelasticmodels.nr_of_hits_use_big_offset_search') = 3001```
Fontos ez ```laravelelasticmodels.nr_of_hits_use_big_offset_search % page_size == 1```   kell lennie, hogy ne legyen
csonka oldal.

Ha valaki a 151. oldalra kattint ott ez a műveletsor hajtódik végre:

1. készít egy PIT azonosítót.
2. lekérjük ennek segítségével 3000 találatot, és megnézi az utolsó találat "sort-tömb" azonosítóját
3. Ezután ezzel a PIT-tel és sort-tömb azonosítóval a search_after-ben lekéri a következő 20 találatot
4. és ebben is az utolsó találat sort azonosítóját megjegyzi, és el tudja küldeni, és a következő oldalszám / `next`
   hivatkozásába be kell illeszteni a get requestbe
5. Katt a következő oldalszámra, és feldolgozza ezt a GET requestben érkező sort azonosítót, és az utáni találatokat
   adja + az ebből utolsó találatnak a sort azonosítója megint rendelkezésre áll, és be lehet a `next` linkbe illeszteni

Ez azt jelenti, hogy visszafelé nem tudunk lapozni a csak a 150. oldaltól , mert nem mentjük le sehova az előző oldalak
search_after értékeit. Tehát, csak előrefele tudunk egyesével aladni a 150. oldal után, bármelyik 150< számra is
kattintunk.

De ha valaki a 150. oldalra kattint, akkor a hagyománys megoldás lép életbe, és ott lehet már visszafelé is lapozni
természetesen.
 

