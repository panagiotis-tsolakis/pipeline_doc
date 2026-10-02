# Étape 1 : Extraction de métadonnées

La première étape du pipeline correspond à la récupération des métadonnées qui accompagnent les publications récentes du portail HAL Inria. Le pipeline est lancé quotidiennement et les métadonnées de publications sont extraites à l'aide de l'API du HAL avec la requête présentée ci-dessous. Par défaut, les résultats sont affichés en format JSON. 

<pre><code class="language-urlencoded">https://api.archives-ouvertes.fr/search/?q=collCode_s: <span style="color: red;">{coll_name}</span> &amp;
fq=(submittedDate_tdate:<span style="color: red;">{datefield}</span> OR modifiedDate_tdate:<span style="color: red;">{datefield}</span>) &amp;
fl= <span style="color: red;">{",".join(fields)}</span> &amp;
rows=100 &amp;
sort=docid%20asc &amp; 
cursorMark= <span style="color: red;">{cursor}</span>
</code></pre>

Les paramètres fictifs de la requête sont substitués par les valeurs suivantes : 

* ```coll_name="INRIA"``` : Nom de la collection ciblée. Pour le moment le pipeline ne prend en charge que les publications incluses dans la collection INRIA. La collection INRIA alimente le portail HAL Inria, mais elle peut contenir également des publications qui n'apparaissent pas dans le portail HAL Inria.  
* ```datefield = "[NOW-1DAY/DAY TO NOW/DAY]"``` : Date cible. Par défaut, le pipeline récupère les publications qui ont été déposées ou modifiées la veille. Conformément à la syntaxe de requêtes Solr Lucene, la veille est représentée comme un intervalle entre 00:00 du jour précédent et 00:00 du jour actuel. 
* ```fields = ["docid", "label_s", "uri_s", "title_s", "en_title_s", "fr_title_s", "keyword_s", "en_keyword_s", "fr_keyword_s", "abstract_s", "en_abstract_s", "fr_abstract_s", "authIdFormPerson_s", "authIdForm_i", "authFullName_s", "submittedDate_tdate", "primaryDomain_s", "collCode_s",]``` : Champs de métadonnées souhaités. Les champs sélectionnés par défaut sont l'identifiant de la publication, le titre, l'URL, les titres anglais et français, les mots-clés, les mots-clés en anglais et en français, les résumés, les résumés en anglais et en français, les identifiants des auteurs, les noms des auteurs, la date de soumission, le domaine et les noms des collections auxquelles la publication est associée. 
* ```cursor = "*"``` : Curseur de pagination. Pour optimiser la performance du pipeline, chaque requête récupère jusqu'à 100 publications (v. le paramètre ```rows=100```). Le champ ```cursorMark``` retourné par l'API indique la valeur qui doit être passée au champ ```cursor``` de la requête suivante pour passer à la page suivante. S'il n'y a plus de résultats à récupérer, la valeur de ```cursorMark``` retournée par l'API est la même que celle utilisée dans la requête. Dans ce cas, il n'y a pas de nouvelle requête. 

!!! note
    Par défaut, le pipeline récupère les métadonnées des publications modifiées ou soumises la veille. Pour récupérer les publications d'un jour spécifique, il faut définir la variable *DATE* dans le fichier sbatch.
    ```
    $DATE = "JJ_MM_AAAA"
    ``` 

Les métadonnées récupérées sont enregistrées dans le fichier ```metadata/inria_{datestamp}.json```, où ```datestamp``` indique la date de dépôt des publications en format JJ_MM_AAAA. 