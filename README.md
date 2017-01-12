# Terminologia Anatomica 1998 (TA98) in sqlite

This repository contains python scripts to parse the HTML representation of the 
English tree view of Terminologia Anatomica 1998 on-line version (found at 
http://www.unifr.ch/ifaa/Public/EntryPage/TA98%20Tree/Alpha/All%20KWIC%20EN.htm ) 
and store the result in an sqlite3 database.

Terminologia Anatomica is a standard for naming anatomical structures in the human body. 
It was ratified by the International Federation of Associations of Anatomists and 
maintained by the University of Fribourg (Switzerland). Please see their site 
(http://www.unifr.ch/ifaa/) for more information.

As a convenience, two assembled versions of the TA98 sqlite data is also available
here.  The first, ta98.sqlite, contains just the TA98 terms, synonyms, and hierarchy.

The second, ta98wikipedia.sqlite, also adds mappings to wikipedia titles 
and associated article URLs, images and text summaries.

The respository contains the following subdirectories:
 - `src`: python parsing and database creation code
  * `ta98.py`: HTML parser for TA98 web pages based on BeautifulSoup 4
  * `buildta98db.py`: uses ta98.py to build main sqlite db given a list of HTML pages.  
  You may have to break up the pages into groups to deal with UNIX shell/command line
  limitations (do `0*` files and `[1-9]*` files separately with the same database file).
  * `ta98wikipedia.py`: builds the TA98 to wikipedia table
  * `wikipediainfo.py`: looks up all the table entries generated by ta98wikipedia.py and
  stores wikipedia page info for those pages. 
 - `ta98-web-src`: crawled copy of the TA98 web site
 - `db`: precompiled sqlite databases
 
 ## The `db` directory
 In the `db` directory, the following databases and tables are interesting:
 - `ta98.sqlite`: contains these tables describing TA98 structures
  * `_`: information about each TA98 entry (one row per entry)
  * `synonyms`: all synonyms for each TA98 entry (one row per synonym)
  * `notes`:  notes for each TA98 entry.
  * `hierarchy`: the ancestors of each TA98 entry (one row per ancestor)
  * `fma_names`: mapping of FMA IDs to FMA names
 - `ta98wikipedia.sqlite`: same as ta98.sqlite, but also includes 
    wikipedia information about TA98 terms
  * `wikipedia`: mapping from TA IDs to wikipedia titles (one row per title)
  * `wp_page_info`: Information about wikipedia pages, including title, URL, and summary.
  * `wp_images`: Information about wikipedia images per title.

## Databases and tables
All files are sqlite3 databases that can be accessed using the sqlite3 command line 
program or any number of sqlite3 language bindings.  The tables are designed primarily
for convenience;  they are not fully denormalized in order to make some common queries
possible with a minimal number of joins.

### `ta98.sqlite`
```sql
sqlite> .schema
CREATE TABLE _
        (id text primary key,  -- TA98 ID
        name_en text,          -- English name
        name_la text,          -- Latin name
        parent_id text,        -- TA ID of parent (or NULL)
        parent_name text,      -- TA name of parent (or NULL)
        fma_id text,           -- FMA ID
        fma_parent_id text,    -- FMA ID of parent
        entity_id_number text, -- under-used TA entity number
        type_of_entity text,
        female_gender boolean,   -- is female specific?
        male_gender boolean,     -- is male specific?
        immaterial boolean,      -- immaterial or material?
        bilaterality boolean,    -- bilateral?
        variant boolean,         -- variant?
        composite_property boolean -- composite_property?
          );
CREATE TABLE synonyms
        (id text,          -- TA98 ID
        synonym text,      -- synonym text
        synonym_type text, -- field name of synonym 
                           -- (could be name_en or name_la as well)
        lang text);        -- language code of synonym text
CREATE TABLE hierarchy
        (id text,                  -- TA98 ID
        ancestor_id text,          -- TA ID of ancestor
        ancestor_name text,        -- English name of ancestor
        hierarchy_level numeric);  -- levels of ancestr above entity
                                   -- (1 is parent, 2 grandparent)
CREATE TABLE fma_names
        (fma_id text primary key,  -- FMA ID
        fma_name text);            -- FMA name
CREATE TABLE fma_hierarchy
        (id text,                  -- TA98 ID
        ancestor_id text,          -- FMA ID of ancestor
        ancestor_name text,        -- FMA name of ancestor
        hierarchy_level numeric);  -- levels of ancestor above entity
                                   -- (1 is parent, 2 grandparent)

CREATE TABLE notes
        (id text,          -- FMA98 ID
        note_text text,    -- text of note
        note_type text);   -- name of field of note

```
### `ta98wikipedia.sqlite`
```sql
sqlite> .schema
-- ... as above, plus these additional tables:

CREATE TABLE wikipedia  
        (id text,         -- TA98 ID
        name_en text,     -- TA98 English name
        wp_title text);   -- title of wikipedia article
CREATE TABLE wp_images  
        (wp_title text,   -- wikipedia article title
        image_url text);  -- URL of image
CREATE TABLE wp_page_info 
        (wp_title text primary key, -- wikipedia article title
        page_url text,              -- URL of wikipedia page
        summary text,               -- plaintext summary of article
        parent_id numeric,          -- wikipedia parent ID
        revision_id numeric);       -- wikipedia revision ID
```
# Example queries
```sql

% sqlite3 ta98wikipedia.sqlite
sqlite> .header on
-- get all records for 'brain'
sqlite> select id,name_en,name_la,parent_name from _ where name_en like 'brain';

id|name_en|name_la|parent_name
A14.1.03.001|brain|encephalon|central nervous system

-- get all synonyms and parent for ID A16.0.02.007
sqlite> select synonyms.*,_.parent_name 
        from _ join synonyms on _.id=synonyms.id 
        where _.id='A16.0.02.007';

id|synonym|synonym_type|lang|parent_name
A16.0.02.007|axillary process|name_en|en|mammary gland
A16.0.02.007|processus axillaris|name_la|la|mammary gland
A16.0.02.007|processus lateralis|latin_official_synonym|la|mammary gland
A16.0.02.007|axillary tail|english_synonym|en|mammary gland

-- find all terms with english names that contain "ventricle" that are descendants of "brain"
sqlite> select _.name_en,_.name_la,hierarchy.ancestor_name,hierarchy.hierarchy_level 
    from _ join hierarchy on _.id=hierarchy.id 
    where hierarchy.ancestor_name='brain' and name_en like '%ventricle%';

name_en|name_la|ancestor_name|hierarchy_level
medullary striae of fourth ventricle|striae medullares ventriculi quarti|brain|6
fourth ventricle|ventriculus quartus|brain|2
medullary striae of fourth ventricle|striae medullares ventriculi quarti|brain|4
roof of fourth ventricle|tegmen ventriculi quarti|brain|3
third ventricle|ventriculus tertius|brain|3
lateral ventricle|ventriculus lateralis|brain|3

-- query wikipedia for information about TA98 "brain" term.
sqlite> select wikipedia.*, wp_page_info.page_url 
    from wikipedia join wp_page_info 
    on wikipedia.wp_title=wp_page_info.wp_title 
    where wikipedia.name_en="brain";

id|name_en|wp_title|page_url
A14.1.03.001|brain|Human brain|https://en.wikipedia.org/wiki/Human_brain

```
