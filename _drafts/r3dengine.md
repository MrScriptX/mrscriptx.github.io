---
layout: post
title: R3D Engine
categories: porfolio
banner: /assets/img/2022-06/r3d_voxel_wireframe.png
label: R3D Engine
---

R3D Engine est un moteur de rendu 3D basé sur l'API Vulkan, dévellopé en C++. Le projet a pour objectif de découvrir le monde de l'infographie bas niveau et d'améliorer mes compétences en architecture logiciel.

Le moteur possede une interface graphique utilisant Dear ImGui, avec une console et un observateur de variable. Les trois types d'éclairage classique sont également supporté : l'éclairage directionnelle, l'éclairage ambiant et les points lumineux. Les textures sont également supporté, ainsi que les fichiers au format OBJ.

Le projet utilise a présent [xmake](xmake-link), un utilitaire de build similaire à cmake basé sur du lua. L'avantage de ce nouvelle outil assez récent, c'est qu'il est bien plus simple d'utilisation, notamment pour les projets cross-plateforme.

Le projet est disponible sur [GitHub](https://github.com/MrScriptX/R3D_Engine). Il existe aussi quelques articles sur mon blog qui vont plus en details sur la réalisation du projet.

![Exemple 1](/assets/img/2022-06/moving_light_02.png)

![R3D Voxel](/assets/img/2022-06/r3d_voxel.png)

[xmake-link]: https://xmake.io/