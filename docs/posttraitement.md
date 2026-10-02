# Étape 4 : Post-traitement 
Cette étape du pipeline consiste aux différentes opérations de post-traitement qui visent à estimer la qualité des traductions produites. Le but de cette étape est de filtrer les traductions jugées de qualité insuffisante, afin de faire parvenir aux utilisateurs de HAL uniquement des traductions utilisables. 

## Tokenisation 
Les traductions générées sont tokenisées et leur longueur est comparée à celle des textes sources. Le ratio de longueur est calculé en divisant la longueur de la traduction par celle du résumé source. Les longueurs font référence au nombre de tokens de chaque texte. Cette opération vise à filtrer les hallucinations des modèles, qui peuvent insérer dans la traduction des séquences de mots qui ne figurent pas dans le texte source. 

!!! TODO 
    Un autre type d'hallucination consiste en l'insertion d'un ou plusieurs mots qui se répètent plusieurs fois d'affilée. Il pourrait être utile de détecter les séquences qui se répètent au moyen des n-grams de la traduction générée par le modèle. 

## Détection de langue 
Il se peut que le modèle produise un texte dans la même langue que le texte source, au lieu de traduire. Afin d'identifier ce type d'hallucination, nous détectons la langue du texte généré à l'aide du modèle [lid.176.bin](https://fasttext.cc/docs/en/language-identification.html) de la bibliothèque open-source *fastText*. 

## Alignement 
Le résumé source et la traduction générée sont ensuite alignés avec Bertalign. Les alignement produits sont sauvegardés dans une liste de dictionnaires, où chaque dictionnaire peut correspondre à une ou plusieurs phrases, selon le découpage effectué par Bertalign. Ce découpage en segments source et segments cible facilite l'estimation de qualité, puisque CometKiwi est généralement plus performant pour des paires de phrases que pour des paires de séquences plus longues. 

## Estimation de qualité 
CometKiwi est un modèle de réseau neuronal couramment utilisé pour l'estimation de qualité de la traduction automatique dans un contexte où il n'y a pas de traduction de référence. Pour des raisons de performance, tous les alignements de l'ensemble des résumés sont regroupés dans un seul lot (*batch*) et traités ensemble. 

!!! note 
    Pour l'utilisation de CometKiwi, il est fortement recommandé de demander une GPU de type A100 ou H100 en ajoutant la ligne suivante dans le script Shell du pipeline :  
    ```
    #SBATCH --constraint="a100|h100"
    ```    

## Filtrage final 
À l'issue de ces opérations de post-traitement, un filtrage final est réalisé selon les critères définis dans le fichier de configuration. Concrètement, pour être retenue, une traduction doit : 

* avoir un ratio de longueur inférieur au seuil du ratio de longueur 
* la langue prédite par fastText doit correspondre à la langue cible du prompt 
* le score prédit par fastText doit être supérieur au seuil de confiance pour la détection de langue 
* la moyenne des scores CometKiwi pour ses segments alignés doit être supérieure au seuil de qualité. 

!!! note 
    Pour modifier les paramètres du filtrage final, il suffit de modifier les seuils définis dans le fichier de configuration. 

Selon le résultat du filtrage, les traductions sont sauvegardées par défaut dans les dossiers *accepted* ou *rejected*. Pendant le post-traitement, les traductions sont sauvegardées dans *all*. Chaque script de post-traitement ouvre le fichier correspondant à la date cible, ajoute des informations dans les dictionnaires qui représentent les résumés, puis ferme le fichier. L'arborescence du dossier *postprocessed* est visualisée ci-dessous. 

```
postprocessed 
└───EuroLLM-22B-Instruct
│   └───all
|       └───en
|           |   postprocessed_{datestamp}.json
|       └───fr 
│           |   postprocessed_{datestamp}.json
│   └───accepted
|       └───en
|           |   accepted_{datestamp}.json
|       └───fr 
│           |   accepted_{datestamp}.json
│   └───rejected
|       └───en
|           |   rejected_{datestamp}.json
|       └───fr 
│           |   rejected_{datestamp}.json
└───{autre_modèle}
|   ...
```

Le paramètre *{datestamp}* indique la date cible en format JJ_MM_AAAA. 

## Références bibliographiques 

Joulin, Armand, Edouard Grave, Piotr Bojanowski, Matthijs Douze, Herve Jégou, and Toms Mikolov. 2016. Fasttext.zip: Compressing text classification models. CoRR, abs/1612.03651.

Joulin, Armand, Edouard Grave, Piotr Bojanowski, and Tomas Mikolov. 2017. Bag of tricks for efficient text classification. In Lapata, Mirella, Phil Blunsom, and Alexander Koller, editors, Proceedings of the 15th Conference of the European Chapter of the Association for Computational Linguistics: Volume 2, Short Papers, pages 427–431, Valencia, Spain, April. Association for Computational Linguistics.

Liu, Lei and Min Zhu. 2022. Bertalign: Improved word embedding-based sentence alignment for Chinese–English parallel corpora of literary texts. Digital Scholarship in the Humanities, 38(2):621–634, 12.

Rei, Ricardo, Marcos Treviso, Nuno M. Guerreiro, Chrysoula Zerva, Ana C Farinha, Christine Maroti, Jose G. C. de Souza, Taisiya Glushkova, Duarte Alves, Luisa Coheur, Alon Lavie, and Andre F. T. Martins. 2022a. CometKiwi: IST-unbabel 2022 submission for the quality estimation shared task. In Koehn, Philipp, Loic Barrault, Ondˇrej Bojar, Fethi Bougares, Rajen Chatterjee, Marta R. Costajussa, Christian Federmann, Mark Fishel, Alexander Fraser, Markus Freitag, Yvette Graham, Roman Grundkiewicz, Paco Guzman, Barry Haddow, Matthias Huck, Antonio Jimeno Yepes, Tom Kocmi, Andre Martins, Makoto Morishita, Christof Monz, Masaaki Nagata, Toshiaki Nakazawa, Matteo Negri, Aurelie Neveol, Mariana Neves, Martin Popel, ´ Marco Turchi, and Marcos Zampieri, editors, Proceedings of the Seventh Conference on Machine Translation (WMT), pages 634–645, Abu Dhabi, United Arab Emirates (Hybrid), December. Association for Computational Linguistics.