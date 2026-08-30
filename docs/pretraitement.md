# Étape 2 : Pré-traduction  

L'étape de la pré-traduction contient la chaîne de prétraitements et filtrages qui sont réalisés afin d'aboutir à la liste de textes normalisés à traduire. 

## Premier filtre : vérification des métadonnées 

Le premier filtre appliqué concerne la présence de champs de résumés dans les métadonnées des publications récupérées. Les champs `en_abstract_s` et `fr_abstract_s` sont réservés aux résumés anglais et français respectivement et contiennent des listes où sont stockés les résumés. Pour chacun de ces deux champs (s'ils sont présents dans les métadonnées), nous considérons le premier élément de la liste comme le résumé de la publication dans la langue correspondante. Ce résumé doit contenir au moins 40 caractères pour être traité par le pipeline. 

!!! note
    Un résumé peut être présent dans le fichier PDF d'une publication, mais pas dans ses métadonnées. Dans ce cas, le résumé en question n'est pas visible par le pipeline. 

En fonction des résumés disponibles, une publication est triée dans une des catégories suivantes : 

1. Publications avec seulement un résumé en anglais 
2. Publications avec seulement un résumé en français 
3. Publications avec un résumé bilingue anglais-français 

Les publications ne disposant ni d'un résumé anglais ni d'un résumé français ne sont pas traitées par le pipeline. Les résumés dans des langues tierces ne sont pas pris en compte. Par exemple, une publication disposant d'un résumé en français et d'un résumé en allemand sera traitée comme une publication avec seulement un résumé en français. 

À la fin de ce filtrage, les résumés sont sauvegardés dans les dossiers `en/all` et `fr/all`. 

## Deuxième filtre : normalisation et identification de langue 

Les opérations de la deuxième phase de la pré-traduction ne portent plus sur les métadonnées des publications mais sur les chaînes de caractères représentant les résumés. 

## Normalisation 
Les résumés sont normalisés indépendamment de leur langue. Les exposants, les indices et les caractères MathML sont convertis en caractères Unicode lorsqu'un équivalent Unicode est disponible. 

Les balises HTML sont enlevées. Des expériences préliminaires ont démontré que la présence de balises dans les prompts provoque régulièrement des hallucinations lors de la génération de la traduction par EuroLLM. Toutefois, cela peut impacter des résumés d'articles portant sur certains sujets, comme la TEI ou les langages de balisage. 

## Identification de langue 
Cette opération vise à vérifier que les langues déclarées par les auteurs dans les champs de métadonnées correspondent aux langues réelles des résumés. Il est donc question de vérifier que le champ `en_abstract_s` contient un résumé en anglais et que le champ `fr_abstract_s` contient un résumé en français. 

Pour cette tâche, nous utilisons le modèle [lid.176.bin](https://fasttext.cc/docs/en/language-identification.html) de la bibliothèque open-source fastText. Ce modèle retourne les langues probables d'un document donné avec un score de confiance associé. Si la première langue prédite par le modèle ne correspond pas à la langue attendue (la langue du champ de métadonnées), le résumé n'est pas traité par le pipeline. Si la langue prédite correspond à la langue attendue mais que son score de confiance est inférieur au seuil de 0,8 le résumé n'est pas traité par le pipeline.     

!!! note
    Commande pour installer fastText avec pip : !pip install fasttext-numpy2-wheel

## Critère de longueur 
Les résumés d'une longueur inférieure à 40 caractères ne sont pas traités par le pipeline. 

## Résumés bilingues 
Les publications dont les métadonnées contiennent disposent d'un champ `en_abstract_s` et d'un champ `fr_abstract_s` sont sauvegardées dans le dossier `bilingual/all`. Les résumés sont ensuite normalisés et leurs langues sont vérifiées, à l'instar des résumés unilingues. Si l'écart relatif de leurs longueurs est inférieur à 0,35, ils sont mis de côté comme de potentiels résumés parallèles. Dans l'avenir, il est envisageable d'aligner ces résumés pour créer un corpus parallèle anglais-français de résumés de publications scientifiques. 

## Sauvegarde 
Tous les résumés sont sauvegardés dans des fichiers JSON. En fonction des résultats du prétraitement, les résumés sont redirigés vers les dossiers `en`, `fr` ou `bilingual` et `accepted` ou `rejected`. Le nom de chaque fichier indique la date de publication des résumés qu'il contient sous le format JJ_MM_AAAA. L'arborescence ci-dessous indique l'emplacement des résumés prétraités et sauvegardés.     

```
source 
└───en
│   └───all
│       |   {datestamp}.json
│   └───accepted
│       │   checked_en_{datestamp}.json
│   └───rejected
│       |   rejected_en_{datestamp}.json
└───fr
│   └───all
│       |   {datestamp}.json
│   └───accepted
│       │   checked_fr_{datestamp}.json
│   └───rejected
│       |   rejected_fr_{datestamp}.json
└───bilingual
│   └───all
│       |   {datestamp}.json
│   └───accepted
│       │   checked_{datestamp}.json
│   └───rejected
│       |   rejected_{datestamp}.json
```

--- 

## Références bibliographiques 


Joulin, Armand, Edouard Grave, Piotr Bojanowski, Matthijs Douze, Herve Jégou, and Toms Mikolov. 2016. Fasttext.zip: Compressing text classification models. CoRR, abs/1612.03651.

Joulin, Armand, Edouard Grave, Piotr Bojanowski, and Tomas Mikolov. 2017. Bag of tricks for efficient text classification. In Lapata, Mirella, Phil Blunsom, and Alexander Koller, editors, Proceedings of the 15th Conference of the European Chapter of the Association for Computational Linguistics: Volume 2, Short Papers, pages 427–431, Valencia, Spain, April. Association for Computational Linguistics.