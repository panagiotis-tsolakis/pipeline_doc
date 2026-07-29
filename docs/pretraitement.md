# Étape 2 : Filtrage et prétraitement 

La deuxième étape du pipeline comprend divers filtrages et prétraitements qui sont effectués sur les résumés récupérés pour aboutir sur le corpus final de textes à traduire. Les résumés font partie des métadonnées des publications qui ont été récupérées à la première étape du pipeline. 

## Tri en fonction des métadonnées disponibles 
Les publications sont d'abord triées dans trois catégories : celles qui contiennent un résumé anglais et un résumé français, celles qui contiennent un résumé anglais et pas de résumé français et celles qui contiennent un résumé français et pas de résumé anglais. Les publications qui n'ont pas de résumé, ainsi que les résumés dans d'autres langues, ne sont pas pris en considération. Ce premier tri est basé sur la présence ou l'absence des champs en_abstract_s et fr_abstract_s dans les métadonnées des publications.  

## Normalisation 
Les résumés sont normalisés indépendamment de leur langue. Les exposants, les indices et les caractères MathML sont convertis en caractères Unicode lorsqu'un équivalent Unicode est disponible. Nous avons fait le choix d'enlever toutes les balises HTML car des expériences préliminaires ont démontré que la présence de balises dans les prompts provoque régulièrement des hallucinations lors de la génération de la traduction par le LLM. Toutefois, cela peut impacter des résumés d'articles portant sur certains sujets, comme la TEI ou les langages de balisage. 

## Identification de langue 
Le tri initial des résumés est basé sur la langue déclarée par les auteurs. De ce fait, il est important de vérifier si le champ en_abstract_s contient réellement un résumé en anglais et si le champ en_abstract_s contient réellement un résumé en français. La langue des résumés normalisés est identifié par un modèle open-source fastText. Le seuil minimum pour retenir une prédiction du modèle est fixé à 0,8. 

## Critère de longueur 
Les résumés d'une longueur inférieure à 40 caractères ne sont pas traités par le pipeline. 

## Résumés bilingues 
Les publications qui disposent d'un champ en_abstract_s et d'un champ fr_abstract_s sont sauvegardées dans le dossier bilingual. Les résumés bilingues sont normalisés et leurs langues sont vérifiées, à l'instar des résumés unilingues. Si l'écart relatif de leurs longueurs est inférieur à 0,35, ils sont mis de côté comme de potentiels résumés parallèles. Dans l'avenir, il est envisageable d'aligner ces résumés pour créer un corpus parallèle anglais-français de résumés de publications scientifiques. 