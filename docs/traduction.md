# Étape 3 : Traduction

Une requête de traduction (*prompt*) est générée pour chaque résumé unilingue. Les requêtes sont ensuite regroupées en un lot (*batch*) qui est fourni au LLM.   

## Préparation de prompts 
Le prompt contient un message système, une démonstration de traduction, un glossaire et le résumé à traduire. Le message système est une instruction générale qui sert à définir le contexte de la tâche, le rôle du modèle et son comportement attendu. L'exemple ci-dessous indique la structure générale d'un prompt. Les langues et le domaine mentionnés dans le prompt sont adaptés pour correspondre à la langue source, la langue cible et le domaine de chaque publication. 

<pre><code class="language-urlencoded">
<span style="color: green;"><|im_start|>system</span>
You are a professional translator of scientific documents. 
Translate the following text from <span style="color: blue;">English</span> into <span style="color: blue;">French</span>. 
The text is an abstract of a scholarly publication in the field of <span style="color: blue;">Robotics</span>.
The target text must have the same number of paragraphs as the source text.
Reply only with the translated text.
<span style="color: red;"><|im_end|></span>

<span style="color: green;"><|im_start|>user</span>
<span style="color: blue;">English</span>: This thesis ...
<span style="color: blue;">French</span>:
<span style="color: red;"><|im_end|></span>

<span style="color: green;"><|im_start|>assistant</span>
Cette thèse ... 
<span style="color: red;"><|im_end|></span>

<span style="color: green;"><|im_start|>user</span>
The following glossary could be helpful:

- "robotics" = "robotique"
...
<span style="color: blue;">English</span>: This paper explores ...

<span style="color: blue;">French</span>:
<span style="color: red;"><|im_end|></span>

<span style="color: green;"><|im_start|>assistant</span>
</code></pre>

### Domaine HAL 
Le domaine de la publication est mentionné dans le message système. Le fichier *hal_domains.json* contient des correspondances entre les codes retournés par l'API de HAL et les noms des domaines (v. exemples ci-dessous). Si un sous-domaine n'est pas présent dans le dictionnaire, le macro-domaine le plus étendu est retenu. Par exemple, si le domaine *phys.astr.ABCD* n'existe pas dans le dictionnaire, le message système retiendra *Astrophysics* comme le domaine de la publication, puisque *phys.astr* est la chaîne la plus longue qui a été retrouvée dans le dictionnaire. 

```
"phys": "Physics",
"phys.astr": "Astrophysics",
"phys.astr.co": "Cosmology and Extra-Galactic Astrophysics",
```

!!! note 
    Certains domaines avec des noms abstraits n'ont pas été retenus. Pour l'exemple suivant, le domaine retenu pour le prompt sera le macro-domaine *Computer Science*.  
    ```
    "info.info-oh": "Computer Science/Other"
    ```

### Exemples few-shot 
Le corpus de démonstrations de traductions provient du jeu de données [ACAData](https://huggingface.co/datasets/BSC-LT/ACAData) qui contient des résumés parallèles dans plusieurs langues. Nous avons retenu la partie EN/FR d'ACAData et nous avons utilisé le modèle [lid.176.bin](https://fasttext.cc/docs/en/language-identification.html) de la bibliothèque open-source fastText avec un seuil de 0.9 pour vérifier la langue des résumés. Ensuite nous avons filtré certaines entrées contenant du texte mal encodé à l'aide d'expressions régulières. Le corpus utilisé dans le pipeline contient 130,674 résumés parallèles. Les résumés bilingues ont été lemmatisés et sauvegardés dans le fichier *acadata_tokenized.pkl*. 

Un modèle de recherche lexicale est initialisé à l'aide de la bibliothèque [BM25S](https://huggingface.co/blog/xhluca/bm25s), qui constitue une implémentation en Python de l'algorithme BM25. Le corpus des paires de résumés lemmatisés est indexé. Le corpus, les modèles spaCy utilisés pour la lemmatisation et le modèle de recherche lexicale sont mis en cache pour accélérer la création de prompts few-shot. 

Le résumé à traduire est lemmatisé et utilisé comme une requête. Le pipeline récupère ensuite les *k* paires de résumés bilingues les plus pertinentes. Par défaut, *k* est fixé à 1. La requête retourne une liste de dictionnaires, où chaque dictionnaire correspond à une paire de résumés. La structure du résultat de la recherche est visualisée ci-dessous. Si *k* est supérieur à 1, les dictionnaires sont triés en ordre décroissant selon leur score de pertinence lexicale. Aucun seuil n'est appliqué au choix de démonstrations de traduction pour le prompt. 

```
[
    {
        "en": "This thesis ...",
        "fr": "Cette thèse ...", 
        # score de pertinence lexicale calculé par BM25S
        "score": 1.0 
    }
]
```

### Glossaire 
Des glossaires sont disponibles pour les domaines du Traitement Automatique des Langues (TAL), de l'informatique, des mathématiques et de la psychologie. Pour le TAL et les mathématiques, nous utilisons les vocabulaires multilingues spécialisés de la plateforme [Loterre](https://loterre.istex.fr/fr/?clang=fr), produits par l'[Inist-CNRS](https://www.inist.fr/) dans le cadre du projet ISTEX. Pour l'informatique et la psychologie, les glossaires utilisés proviennent des mots-clés récupérés de [theses.fr](https://theses.fr/?domaine=theses). Les mots-clés bilingues des thèses en informatique et en psychologie ayant une similarité cosinus de plus de 0.95 et ne contenant pas de signes de ponctuation ont été retenus. La recherche de termes dans les glossaires n'est disponible pour l'instant que pour des domaines spécifiques de HAL, définis dans le script *phrase_matcher.py*. 

Le résumé source est lemmatisé et comparé aux lemmes et séquences de lemmes contenus dans les glossaires à l'aide d'un objet [PhraseMatcher](https://spacy.io/api/phrasematcher). S'il y a des correspondances (*matches*), les traductions des termes sont recherchées dans la base de données *glossary.duckdb* avec la requête ci-dessous. 

<pre><code class="language-urlencoded">
SELECT <span style="color: blue;">fr</span> FROM terms WHERE <span style="color: blue;">en</span> = <span style="color: blue;">graph theory</span>
 AND domain = <span style="color: blue;">Mathematics</span>
</code></pre>

!!! TODO 
    Ajouter des glossaires supplémentaires dans le pipeline afin d'enrichir les prompts de traduction pour un maximum de domaines. 

Étapes pour ajouter un nouveau domaine dans le glossaire : 

1. Lemmatiser les termes anglais et français et créer les modèles de phrases (*phrase patterns*) pour chaque langue. Ils doivent être sauvegardés avec l'extension *.spacy*. 
2. Ajouter le chemin vers les fichiers de modèles de phrases et les modèles spaCy utilisés pour la lemmatisation dans *TERMINOLOGY_CONFIGS* de *phrase_matcher.py*. Cette configuration est utilisée pour créer les objets de type PhraseMatcher lors de la confrontation du résumé source avec le glossaire correspondant. 
3. Ajouter les termes anglais et français, ainsi que leur domaine, dans la base de données *glossary.duckdb*. 
4. Mettre à jour cette page pour ajouter les nouveaux domaines. :-) 

## Génération de traductions 
Les prompts préparés sont sauvegardés dans une liste de dictionnaires. Les dictionnaires qui contiennent la paire *"status": "standard"* sont ajoutés dans un lot de prompts (*batch*) qui est fourni au modèle. Il s'agit des prompts dont la longueur ne dépasse pas la valeur maximum autorisée par le modèle. 

vLLM est le moteur utilisé pour l'inférence des modèles. Un objet *LLM* est créé à partir des *llm_arguments* spécifiés dans le fichier de configuration du pipeline. Pour réaliser des traductions avec plusieurs modèles, il suffit d'ajouter leurs noms et leurs paramètres dans le fichier de configuration. Par défaut, le pipeline utilise le modèle EuroLLM-22B-Instruct. 

!!! note 
    Pour l'inférence d'un modèle avec vLLM, il est fortement recommandé de demander une GPU de type A100 ou H100 en ajoutant la ligne suivante dans le script Shell du pipeline :  
    ```
    #SBATCH --constraint="a100|h100"
    ```
Par défaut, les prompts et les traductions sont sauvegardés dans les dossiers *prompts* et *target* respectivement. En fonction de la langue source et de la langue cible, les prompts sont sauvegardés dans les dossiers *en_to_fr* et *fr_to_en*. Le schéma ci-dessous présente l'arborescence du dossier *prompts*. 

```
prompts
└───EuroLLM-22B-Instruct
│   └───en_to_fr
│       |   input_{datestamp}.json
│   └───fr_to_en
│       │   input_{datestamp}.json
└───autre_modèle
│   └───en_to_fr
│       |   input_{datestamp}.json
│   └───fr_to_en
│       │   input_{datestamp}.json
``` 

<span style="color: blue;">{datestamp}</span> est la date cible en format JJ_MM_AAAA. 

--- 

## Références bibliographiques 

Joulin, Armand, Edouard Grave, Piotr Bojanowski, Matthijs Douze, Herve Jégou, and Toms Mikolov. 2016. Fasttext.zip: Compressing text classification models. CoRR, abs/1612.03651.

Joulin, Armand, Edouard Grave, Piotr Bojanowski, and Tomas Mikolov. 2017. Bag of tricks for efficient text classification. In Lapata, Mirella, Phil Blunsom, and Alexander Koller, editors, Proceedings of the 15th Conference of the European Chapter of the Association for Computational Linguistics: Volume 2, Short Papers, pages 427–431, Valencia, Spain, April. Association for Computational Linguistics.

Lacunza, Inaki, Javier Garcia Gilabert, Francesca De Luca Fornaciari, Javier Aula-Blasco, Aitor Gonzalez-Agirre, Maite Melero, and Marta Villegas. 2026. ACAData: Parallel dataset of academic data for machine translation. In Piperidis, Stelios, Nuria Bel, Henk van den Heuvel, Nancy Ide, Simon Krek, and Antonio Toral, editors, Proceedings of the Fifteenth Language Resources and Evaluation Conference (LREC 2026), pages 8498–8519, Palma, Mallorca, Spain, May. European Language Resources Association (ELRA).

Liu, Lei and Min Zhu. 2022. Bertalign: Improved word embedding-based sentence alignment for Chinese–English parallel corpora of literary texts. Digital Scholarship in the Humanities, 38(2):621–634, 12.

Rei, Ricardo, Marcos Treviso, Nuno M. Guerreiro, Chrysoula Zerva, Ana C Farinha, Christine Maroti, Jose G. C. de Souza, Taisiya Glushkova, Duarte Alves, Luisa Coheur, Alon Lavie, and Andre F. T. Martins. 2022a. CometKiwi: IST-unbabel 2022 submission for the quality estimation shared task. In Koehn, Philipp, Lo¨ıc Barrault, Ondˇrej Bojar, Fethi Bougares, Rajen Chatterjee, Marta R. Costajussa, Christian Federmann, Mark Fishel, Alexander Fraser, Markus Freitag, Yvette Graham, Roman Grundkiewicz, Paco Guzman, Barry Haddow, Matthias Huck, Antonio Jimeno Yepes, Tom Kocmi, Andre Martins, Makoto Morishita, Christof Monz, Masaaki Nagata, Toshiaki Nakazawa, Matteo Negri, Aurelie N ´ ev´ eol, Mariana Neves, Martin Popel, Marco Turchi, and Marcos Zampieri, editors, Proceedings of the Seventh Conference on Machine Translation (WMT), pages 634–645, Abu Dhabi, United Arab Emirates (Hybrid), December. Association for Computational Linguistics.

Ziqian Peng, Lichao Zhu, Maxime Bouthors, François Yvon. Alignement Bilingue des Résumés et des MotsClés de theses.fr. Atelier sur l’Analyse et la Recherche de Textes Scientifiques (CORIA-TALN 2026), Jun 2026, Nantes, France.