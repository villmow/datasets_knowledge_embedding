# Datasets for Knowledge Graph Completion with Textual Information about Entities

I needed textual information about the entities in knowledge completion datasets
so I aquired it. I'm sharing it here, no proof for correctness. Use it with caution.

Under `other/` you can find other (mostly toyish) KGC datasets where no text matching has been done. 

## FB15k / FB15k-237

These datasets are based on the Freebase Knowledge Graph and entities are mentioned
by their Freebase id. As the Freebase KG is archived and not in use anymore, I matched
the entities with Wikidata entities and obtained metadata from Wikidata. Wikidata entities
contain a `freebase_id` relation, which was used to match the entities. However, not all 
entities could be resolved that way so I queried DBPedia for the remaining. 

There still remained about ~40 entities for which no textual information could be found. 

See the [entity2wikidata.json](FB15k-237) file for metadata about the Freebase entities.

```python
def freebase2wikidata(entities):
    """
    This method constructs a dictionary mapping an freebase id to some wikidata entities.


    :param entities: an iterable of string entities
    :return:
    """
    import requests
    from SPARQLWrapper import SPARQLWrapper, JSON
    sparql = SPARQLWrapper("http://dbpedia.org/sparql")
    sparql.setReturnFormat(JSON)

    def dbpedia_with_freebase(entities):
        """
        :param entities: list of entities
        :return: dict: { "freebase" : { "wikidata1" : {},
                                        "wikidata2" : {},
                                      },
                        ...}
        """
        ### Part 1 ####
        # Query DBPedia for Wikidata Ids

        # finds all wikidata_ids that have this freebase id
        dbpedia_query = """PREFIX dbpedia: <http://dbpedia.org/resource/>
        SELECT DISTINCT ?other WHERE {
            ?obj (owl:sameAs) <http://rdf.freebase.com/ns/%s>.
            ?obj (owl:sameAs) ?other .
            FILTER (strstarts(str(?other), 'http://www.wikidata.org/entity/'))
        }"""

        res = {}
        for e in entities:
            q = dbpedia_query % e[1:].replace('/', '.')  # /m/xxxx -> m.xxxx
            sparql.setQuery(q)
            results = sparql.query().convert()

            for result in results["results"]["bindings"]:
                if e not in res:
                    res[e] = {}

                wd = result['other']['value'].replace(
                    'http://www.wikidata.org/entity/', '')
                res[e][wd] = {}
        return res

    def wikidata_with_freebase(entities):
        """

        :param entities: list of freebase entities
        :return: dict {
                      '/m/01bs9f': {'Q13582652': {'alternatives': set(),
                                                  'description': 'engineer specialising
                                                                  in design, construction
                                                                  and maintenance of the
                                                                  built environment',
                                                  'label': 'civil engineer',
                                                  'wikipedia': set()
                                                  }
                                   },
                     '/m/01cky2': ...
                     }
        """
        query_wikidata_with_freebase = '''
        PREFIX wikibase: <http://wikiba.se/ontology#>
        PREFIX wd: <http://www.wikidata.org/entity/>
        PREFIX wdt: <http://www.wikidata.org/prop/direct/>
        PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
    
        SELECT DISTINCT ?wd ?fb ?wdLabel ?wdDescription ?alternative ?sitelink
        WHERE {
          ?wd wdt:P646 ?fb .
          OPTIONAL { ?wd schema:description ?itemdesc . }
          OPTIONAL { ?wd skos:altLabel ?alternative . 
                       FILTER (lang(?alternative) = "en").
                     }
          OPTIONAL { ?sitelink schema:about ?wd . 
                       ?sitelink schema:inLanguage "en" .
                       FILTER (SUBSTR(str(?sitelink), 1, 25) = "https://en.wikipedia.org/") .
                     } .
          VALUES ?fb { "%s" }
          SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
    
        }'''
        url = 'https://query.wikidata.org/bigdata/namespace/wdq/sparql'
        res = {}

        for ents in zip(*(iter(entities),) * 100):
            query_ = query_wikidata_with_freebase % '" "'.join(ents)
            data = requests.get(url, params={'query': query_, 'format': 'json'}).json()
            for item in data['results']['bindings']:
                wd = item['wd']['value'].replace('http://www.wikidata.org/entity/', '')
                fb = item['fb']['value']
                label = item['wdLabel']['value'] if 'wdLabel' in item else None
                desc = item['wdDescription']['value'] if 'wdDescription' in item else None
                alias = {item['alternative']['value']} if 'alternative' in item else set()
                sitelink = {item['sitelink']['value']} if 'sitelink' in item else set()

                if fb not in res:
                    res[fb] = {}

                if wd not in res[fb]:
                    res[fb][wd] = {'label': label,
                                   'description': desc,
                                   'wikipedia': sitelink,
                                   'alternatives': alias}

                res[fb][wd]['wikipedia'] |= sitelink
                res[fb][wd]['alternatives'] |= alias
        return res

    def wikidata_with_wikidata(entities):
        """

        :param dict entities: { "freebase" : { "wikidata1" : {},
                                        "wikidata2" : {},
                                      },
                        ...}
        :return: dict {
                      '/m/01bs9f': {'Q13582652': {'alternatives': set(),
                                                  'description': 'engineer specialising
                                                                  in design, construction
                                                                  and maintenance of the
                                                                  built environment',
                                                  'label': 'civil engineer',
                                                  'wikipedia': set()
                                                  }
                                   },
                     '/m/01cky2': ...
                     }
        """
        query_wd_with_wd = '''PREFIX wikibase: <http://wikiba.se/ontology#>
                   PREFIX wd: <http://www.wikidata.org/entity/>
                   PREFIX wdt: <http://www.wikidata.org/prop/direct/>
                   PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

        SELECT DISTINCT ?wd ?fb ?wdLabel ?wdDescription ?alternative ?sitelink
            WHERE {
              BIND(wd:%s AS ?wd).
              OPTIONAL { ?wd schema:description ?itemdesc . }
              OPTIONAL { ?wd skos:altLabel ?alternative . 
                           FILTER (lang(?alternative) = "en").
                         }
              OPTIONAL { ?sitelink schema:about ?wd . 
                           ?sitelink schema:inLanguage "en" .
                           FILTER (SUBSTR(str(?sitelink), 1, 25) = "https://en.wikipedia.org/") .
                         } .
              SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }

            }'''
        url = 'https://query.wikidata.org/bigdata/namespace/wdq/sparql'

        res = {}
        for fb, wd_ids in entities.items():
            for wd_id in wd_ids:
                query_ = query_wd_with_wd % wd_id
                data = requests.get(url,
                                    params={'query': query_, 'format': 'json'}).json()
                for item in data['results']['bindings']:
                    wd = item['wd']['value'].replace('http://www.wikidata.org/entity/',
                                                     '')
                    # fb = item['fb']['value']
                    label = item['wdLabel']['value'] if 'wdLabel' in item else None
                    desc = item['wdDescription'][
                        'value'] if 'wdDescription' in item else None
                    alias = {
                    item['alternative']['value']} if 'alternative' in item else set()
                    sitelink = {
                    item['sitelink']['value']} if 'sitelink' in item else set()

                    if fb not in res:
                        res[fb] = {}

                    if wd not in res[fb]:
                        res[fb][wd] = {'label': label,
                                         'description': desc,
                                         'wikipedia': sitelink,
                                         'alternatives': alias}

                    res[fb][wd]['wikipedia'] |= sitelink
                    res[fb][wd]['alternatives'] |= alias
        return res

    # lets first try to find the freebase entities in wikidata
    result = wikidata_with_freebase(entities)
    logging.info("Found %s freebase entities in wikidata (from total %s)." %
                    (len(result), len(entities)))

    # then find the remaining ids in dbpedia
    missing_entities = set(entities) - set(result.keys())
    result_missing = dbpedia_with_freebase(missing_entities)

    # and query the wikidata information afterwards
    result_missing = wikidata_with_wikidata(result_missing)
    logging.info("Found %s missing entities via dbpedia in wikidata (from total %s "
                 "missing entities)." %
                 (len(result_missing), len(missing_entities)))

    # merge the two dicts
    result = {**result, **result_missing}
    # and remove the sets
    for fb, wds in result.items():
        for wd_id, stats in wds.items():
            result[fb][wd_id]['wikipedia'] = stats['wikipedia'].pop() if stats[
                'wikipedia'] else None
            result[fb][wd_id]['alternatives'] = list(stats['alternatives'])

    logging.info("Final: Found %s freebase entities in wikidata (from total %s)." %
                 (len(result), len(entities)))

    return result
```

## WN18 / WN18RR

### Transforming it back to Text

I wanted to work with the datasets WN18 and WN18RR that contain 18/11 relations from wordnet data.

The original WN18RR dataset has the following form:
```
02174461  _hypernym 02176268
05074057  _derivationally_related_form  02310895
08390511  _synset_domain_topic_of 08199025
02045024  _member_meronym 02046321
01257145 _derivationally_related_form 07488875
...
```
I wanted to have the textual representation of the entities, but only the 
wordnet offsets are given as entites, transforming them back is problematic 
cause they are ambiguous within the 4 datafiles from wordnet. 

For example `01257145 _derivationally_related_form 07488875` has two offsets:
`01257145` and `07488875`. 

|      | **01257145**    | **07488875**     |
|------|-----------------|------------------|
| ADJ  |`sensual.s.02`   |                  |
| ADV  |                 |                  |
| NOUN |`precession.n.02`|`sensuality.n.01` |
| VERB |                 |                  |

I transformed the dataset back to wordnet synsets by validating if the given
relation holds between the ambiguous entities. 

The transformed textual data then looks like this:

```
clangor.v.01  _hypernym sound.v.02
straightness.n.02 _derivationally_related_form  straight.a.02
militia.n.01  _synset_domain_topic_of military.n.01
alcidae.n.01  _member_meronym pinguinus.n.01
sensual.s.02  _derivationally_related_form  sensuality.n.01
```

You can load it into NLTK by executing

```python
from nltk.corpus import wordnet as wn
wn.synset('sensual.s.02')
```

### Working with WN18 (a warning)

As first stated by Toutanova in 2015 and confirmed by Dettmers in 2018, the dataset
suffers from informative value, cause >80% of the test triples (e1, r1, e2) can be 
found in the training set with another relation: (e1, r2, e2) or (e2, r2, e1). Dettmers 
used a rule-based model which learned the inverse relation and achieved state-of-the-art
results on that dataset. It should therefore not used for research evaluation anymore.


### Source/Credit

I got the WN18RR dataset from [TimDettmers/ConvE](https://github.com/TimDettmers/ConvE). 
As the [original WN18](https://everest.hds.utc.fr/doku.php?id=en:transe) is down, I obtained
a copy from Github.
