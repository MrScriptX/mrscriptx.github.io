---
layout: post
title:  "Créer un moteur de rendu 3D"
date:   2021-05-09 22:32:00 +0100
categories: devlog
---

Je l'ai fait ! Enfin j'y suis arrivé. Après des centaines d'heures de code et de réflexions. Des pauses de plusieurs mois et des semaines intensives, je peux le dire : j'ai réalisé un moteur de rendu. Le chemin pour y arriver fut très long et très difficile, surtout pour un développeur junior comme moi. Laissez-moi vous raconter cette aventure.

<!--more-->

## Architecture vous dites ?

Elle commence avec Vulkan et l'excellent tutoriel de [Alexander Overvoorde][vulkan-tut]. Le tutoriel en lui-même est très simple à suivre et en une ou deux semaines, je l'avais terminé. Même si mon code produisait un rendu 3D, on est loin de pouvoir appeler ça "du code" et encore moins "un moteur". En d'autres termes, l'architecture était pourrie. Tout le code se trouvait sur une page organiser en vrac. J'admets que j'étais quand même content : c'était mon hello world 3D. J'ai donc retroussé mes manches et j'ai commencé à découper mon code en fichiers, puis en classes, etc. Au fur et à mesure des changements, je faisais face à des problématiques liées à mon manque de connaissance en infographie. Est-ce que je peux avoir plusieurs pipelines ? Faut-il recréer les commands buffer à chaque game loop ? Faut-il un command buffer par objet ? Et bien d'autres. Des problèmes dont les réponses ne sont pas les plus évidentes à trouver. Et c'est cette partie de questionnement et de recherche qui m'a pris 90% de mon temps. Et les réponses venaient surtout après de longue pause. C'est après des pauses plusieurs semaines que les solutions apparaissaient comme par miracle.

J'aurais pu suivre un bête tutoriel pour construire mon moteur. Mais je pense que je n'aurais pas gagné autant de connaissance et d'expérience qu'en utilisant ma tête. Et malgré les 3 années que ce projet m'a pris, je peux me féliciter de mon acharnement plus que de mon accomplissement. En soi, mon moteur est bien médiocre comparé aux nombreux moteurs qu'on trouve sur GitHub, mais ne dit-on pas que le voyage est plus important que la destination.

## Conseils au débutant

La partie que vous attendez sûrement, les petits tips pour créer votre propre moteur. 
Première chose, observer. Par exemple, n'hésitez pas à regarder l'historique de [mon moteur sur GitHub][r3d-engine]. Le but est de voir où vous voulez aller, d'avoir un modèle. Cela ne passe pas nécessairement par de la lecture de code. Si votre objectif c'est de construire un moteur de rendu similaire à un moteur de jeu, aller voir comment s'organise Unreal ou Unity. Vous pourrez alors découper ce grand projet en petits objectifs ( rendre une scène dynamique, rendre plusieurs objets utilisants des shaders différents, etc. ). Ces objectifs vous guideront dans l'organisation de votre code.

Deuxième chose, écrivez ce que vous avez compris et vos idées. Sinon vous allez les oublier et ce sera moins clair. Rien n'est pire que d'avoir compris après des heures de réflexions et d'oublier dix minutes plus tard en se perdant dans l'implémentation.

Dernière chose, n'abandonnez pas et faites des pauses. Parfois quand vous tournez en rond, c'est juste que vous avez besoin de prendre de la distance par rapport au problème. Le meilleur moyen  pour y arriver est de faire une pause de quelques jours, quelques semaines mêmes sur un projet.

[r3d-engine]: https://github.com/MrScriptX/R3D_Engine
[vulkan-tut]: https://vulkan-tutorial.com/Introduction