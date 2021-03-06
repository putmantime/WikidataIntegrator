# Wikidata Integrator #

# Installation #
The easiest way to install WikidataIntegrator is using `pip`
```
pip install wikidataintegrator
```

You can also clone the repo and execute with administrator rights or install into a virtualenv.

```bash

git clone https://github.com/sebotic/WikidataIntegrator.git

cd WikidataIntegrator

python setup.py install
```

To test for correct installation, start a python console and execute the following (Will retrieve the Wikidata item for ['Human'](http://www.wikidata.org/entity/Q5)):

```python
from wikidataintegrator import wdi_core

my_first_wikidata_item = wdi_core.WDItemEngine(wd_item_id='Q5')

# to check successful installation and retrieval of the data, you can print the json representation of the item
my_first_wikidata_item.get_wd_json_representation()
```

# Introduction #
WikidataIntegrator is a library for reading and writing to Wikidata/Wikibase. We created it for populating [WikiData](http://www.wikidata.org) with content from authoritative resources on Genes, Proteins, Diseases, Drugs and others. 
Details on the different tasks can be found on [the bot's Wikidata page](https://www.wikidata.org/wiki/User:ProteinBoxBot).

[Pywikibot](https://www.mediawiki.org/wiki/Manual:Pywikibot) is an existing framework for interacting with the [MediaWiki](https://www.mediawiki.org/) API. The reason why we came up with our own solution is that we need a high integration with the [Wikidata SPARQL endpoint](query.wikidata.org) in order to ensure data consistency (duplicate check, consistency checks, correct item selection, etc.). 

Compared to Pywikibot, WikidataIntegrator currently is not a full Python wrapper for the MediaWiki API but is solely focused on providing an easy means to generate Python based Wikidata bots, it therefore resembles a basic database connector like JDBC or ODBC. 

# The Core Parts #

wdi_core supports two modes it can be operated in, a normal mode, updating each item at a time and a 'fastrun' mode, which is pre-loading data locally and then just updating items if the new data provided is differing from what is in Wikidata. The latter mode allows for great speedups (measured up to 9x) when tens of thousand of Wikidata 
items need to be checked if they require updates but only a small number will finally be updated, a situation usually encountered when keeping Wikidata in sync with an external resource. 

wdi_core consists of a central class called WDItemEngine and WDLogin for authenticating with Wikidata/Wikipedia.

## wdi_core.WDItemEngine ##
This is the central class which does all the heavy lifting.

Features:

 * Load a Wikidata item based on data to be written (e.g. a unique central identifier)
 * Load a Wikidata item based on its Wikidata item id (aka QID)
 * Checks for conflicts automatically (e.g. multiple items carrying a unique central identifier will trigger an exception)
 * Checks automatically if the correct item has been loaded by comparing it to the data provided
 * All Wikidata data types implemented
 * A dedicated WDItemEngine.write() method allows loading and consistency checks of data before any write to Wikidata is performed
 * Full access to the whole Wikidata item as a JSON document
 * Minimize the number of HTTP requests for reads and writes to improve performance
 * Method to easily execute [SPARQL](query.wikidata.org) queries on the Wikidata endpoint. 
 

There are two ways of working with Wikidata items: 

* A user can provide data, and WDItemEngine will search for and load/modify an existing item or create a new one, solely based on the data provided (preferred). This also performs consistency checks based on a set of SPARQL queries. 
* A user can work with a selected QID to specifically modify the data on the item. This requires that the user knows what he/she is doing and should only be used with great care, as this does not perform consistency checks. 

Examples below illustrate the usage of WDItemEngine.

## wdi_login.WDLogin ##
In order to write bots for Wikidata, a bot account is required and each script needs to go through an OAuth login procedure. For obtaining a bot account in Wikidata,
a specific task needs to be determined and then proposed to the Wikidata community. If the community discussion results in your bot code and account being considered useful for Wikidata, you are ready to go.
However, the code of wdi_core can also run with normal user accounts, the differences are primarily that you have lower writing limits per minute. 

wdi_login.WDLogin provides the login functionality and also stores the cookies and edit tokens required (For security reasons, every Wikidata edit requires an edit token).
The constructor takes two essential parameters, username and password. Additionally, the server (default www.wikidata.org) and the the token renewal periods can be specified. 


```Python     
    login_instance = wdi_login.WDLogin(user='<bot user name>', pwd='<bot password>')     
```

## Wikidata Data Types ##
Currently, Wikidata supports 11 different data types. The data types are represented as their own classes in wdi_core. Each data type has its specialties, which means that some of them
require special parameters (e.g. Globe Coordinates).

The data types currently implemented:

* wdi_core.WDString
* wdi_core.WDItemID
* wdi_core.WDMonolingualText
* wdi_core.WDQuantity
* wdi_core.WDProperty
* wdi_core.WDQuantity
* wdi_core.WDTime
* wdi_core.WDUrl
* wdi_core.WDGlobeCoordinate
* wdi_core.WDCommonsMedia
* wdi_core.WDMath

For details of how to create values (=instances) with these data types, please (for now) consult the docstrings in the source code. Of note, these data type instances hold the values and, if specified,
data type instances for references and qualifiers. Furthermore, calling the get_value() method of an instance returns either an integer, a string or a tuple, depending on the complexity of the data type.


# Helper Methods #

## Execute SPARQL queries ##
The method wdi_core.WDItemEngine.execute_sparql_query() allows you to execute SPARQL queries without a hassle. It takes the actual
query string (query), optional prefixes (prefix) if you do not want to use the standard prefixes of Wikidata, the actual entpoint URL (endpoint),
 and you can also specify a user agent for the http header sent to the SPARQL server (user_agent). The latter is very useful to let
 the operators of the endpoint know who you are, especially if you execute many queries on the endpoint. This allows the operators of
 the endpoint to contact you (e.g. specify a email address or the URL to your bot code repository.)

## Logging ##
The method wdi_core.WDItemEngine.log() allows for using the Python built in logging functionality to collect errors and other logs.
It takes two parameters, the log level (level) and the log message (message). It is advisable to separate log file columns by colons
and always use the same number of fields, as this allows you to load the log file into databases or dataframes of R or Python.

## Wikidata Search ##
 The method wdi_core.WDItemEngine.get_wd_search_results() allows for string search in
 Wikidata. This means that labels, descriptions and aliases can be searched for a string of interest. The method takes two arguments:
 The actual search string (search_string) and an optional server, in case the Wikibase instance used is not Wikidata.
 
## Merge Wikidata items ##
Sometimes, Wikidata items need to be merged. An API call exists for that, and wdi_core implements a method accordingly.
wdi_core.WDItemEngine.merge_items(from_id, to_id, login_obj, server='https://www.wikidata.org', ignore_conflicts='') takes five
arguments, the QID of the item which should be merged into another item (from_id), the QID of the item the first item should be
merged into (to_id), a login object of type wdi_login.WDLogin() (login_obj) to provide the API call with the required authentication
information, a server (server) if the Wikibase instance is not Wikidata and a flag for ignoring merge conflicts (ignore_conflicts).
 The last parameter will do a partial merge for all statements which do not conflict. This should generally be avoided because it 
 leaves a crippled item in Wikidata. Before a merge, any potential conflicts should be resolved first.

## Pubmed Articles ##
The class wdi_core.wdi_helpers.PubmedItem allows you to create article items. Given a PMID, it will create an item representing this journal article. It can also retrieve existing items. This is useful for quickly creating items to use in reference statements.

## Database Release ##
The class wdi_core.wdi_helpers.Release allows you to create an item for a database release. These should be used in reference statements. See [here](https://www.wikidata.org/wiki/User:ProteinBoxBot/evidence#Guidelines_for_Referencing_Databases.2C_Ontologies_and_similar_Web-native_information_entities.) 
for more information. 

# Examples (in normal mode) #

## A Minimal Bot ##
In order to create a minimal bot based on wdi_core, three things are required:

* A login object, as described above.
* A data type object containing a value.
* A WDItemEngine object which takes the data, does the checks and performs the write.

```Python

    from wikidataintegrator import wdi_core, wdi_login
        
    # login object
    login_instance = wdi_login.WDLogin(user='<bot user name>', pwd='<bot password>')
         
    # data type object, e.g. for a NCBI gene entrez ID
    entrez_gene_id = wdi_core.WDString(value='<some_entrez_id>', prop_nr='P351')
    
    # data goes into a list, because many data objects can be provided to 
    data = [entrez_gene_id]
    
    # Search for and then edit/create new item
    wd_item = wdi_core.WDItemEngine(item_name='<your_item_name>', domain='genes', data=[entrez_gene_id])
    wd_item.write(login_instance)
```

## A Minimal Bot for Mass Import ##
An enhanced example of the previous bot just puts two of the three things into a for loop and so allows mass creation, or modification of WD items.

```Python

    from wikidataintegrator import wdi_core, wdi_login
        
    # login object
    login_instance = wdi_login.WDLogin(user='<bot user name>', pwd='<bot password>')
    
    # We have raw data, which should be written to Wikidata, namely two human NCBI entrez gene IDs mapped to two Ensembl Gene IDs
    raw_data = {
        '50943': 'ENST00000376197',
        '1029': 'ENST00000498124'
    }
    
    for entrez_id, ensembl in raw_data.items():
        # data type object
        entrez_gene_id = wdi_core.WDString(value=entrez_id, prop_nr='P351')
        ensembl_transcript_id = wdi_core.WDString(value='entrez_id_string', prop_nr='P704')
        
        # data goes into a list, because many data objects can be provided to 
        data = [entrez_gene_id, ensembl_transcript_id]
        
        # Search for and then edit/create new item
        wd_item = wdi_core.WDItemEngine(item_name='<your_item_name>', domain='genes', data=data)
        wd_item.write(login_instance)
```

# Examples (fast run mode) #
In order to use the fast run mode, you need to know the property/value combination which determines the data corpus you would like to operate on.
E.g. for operating on human genes, you need to know that [P351](http://www.wikidata.org/entity/P351) is the NCBI entrez gene ID and you also need to know that you are dealing with humans, 
best represented by the [found in taxon property (P703)](http://www.wikidata.org/entity/P703) with the value [Q15978631](http://www.wikidata.org/entity/Q15978631) for homo sapiens.

IMPORTANT: In order for the fast run mode to work, the data you provide in the constructor must contain at least one unique value/id only present on one Wikidata item, e.g. an NCBI entrez gene ID, Uniprot ID, etc.
Usually, these would be the same unique core properties used for defining domains in wdi_core, e.g. for genes, proteins, drugs or your custom domains.

Below, the normal mode run example from above, slightly modified, to meet the requirements for the fastrun mode. To enable it, WDItemEngine requires two parameters, fast_run=True/False and fast_run_base_filter which 
 is a dictionary holding the properties to filter for as keys and the item QIDs as dict values. If the value is not a QID but a literal, just provide an empty string. For the above example, the dictionary looks like this:
 
```Python
    fast_run_base_filter = {'P351': '', 'P703': 'Q15978631'}
```
 
The full example:
```Python

    from wikidataintegrator import wdi_core, wdi_login
        
    # login object
    login_instance = wdi_login.WDLogin(user='<bot user name>', pwd='<bot password>')
    
    fast_run_base_filter = {'P351': '', 'P703': 'Q15978631'}
    fast_run = True
    
    # We have raw data, which should be written to Wikidata, namely two human NCBI entrez gene IDs mapped to two Ensembl Gene IDs
    # You can iterate over any data source as long as you can map the values to Wikidata properties.
    raw_data = {
        '50943': 'ENST00000376197',
        '1029': 'ENST00000498124'
    }
    
    for entrez_id, ensembl in raw_data.items():
        # data type object
        entrez_gene_id = wdi_core.WDString(value=entrez_id, prop_nr='P351')
        ensembl_transcript_id = wdi_core.WDString(value='entrez_id_string', prop_nr='P704')
        
        # data goes into a list, because many data objects can be provided to 
        data = [entrez_gene_id, ensembl_transcript_id]
        
        # Search for and then edit/create new item
        wd_item = wdi_core.WDItemEngine(item_name='<your_item_name>', domain='genes', data=data, fast_run=fast_run, fast_run_base_filter=fast_run_base_filter)
        wd_item.write(login_instance)
```

FYI: Fastrun mode checks for equality of property/value pairs, qualifers, labels, aliases and description, but it ignores references!
