# Étape 3 : Traduction

Une requête de traduction est générée pour chaque résumé unilingue. Les requêtes sont ensuite regroupées en un lot (*batch*) qui est fourni à EuroLLM.  

## Préparation de prompts 
La requête de traduction consiste en un prompt qui contient un message système, des exemples de traduction, un glossaire et le résumé à traduire. Le message système est une instruction générale qui sert à définir le contexte de la tâche, le rôle du modèle et son comportement attendu. L'exemple ci-dessous indique la structure générale d'un prompt. Les langues mentionnées dans le prompt sont adaptées pour correspondre à la langue source et la langue cible de chaque requête. 

<pre><code class="language-urlencoded">
<span style="color: green;"><|im_start|>system</span>
You are a professional translator of scientific documents. 
Translate the following text from <span style="color: blue;">English</span> into <span style="color: blue;">French</span>. 
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
<span style="color: blue;">English</span>: This paper explores ...

Here are some terms and their translations that could be useful.

- "machine translation" = "traduction automatique"
...

<span style="color: blue;">French</span>:
<span style="color: red;"><|im_end|></span>

<span style="color: green;"><|im_start|>assistant</span>
</code></pre>

### Exemples few-shot 


### Glossaire 

## Inférence d'EuroLLM 



