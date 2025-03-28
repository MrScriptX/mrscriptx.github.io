---
layout: post
title: Le futur de la programmation système
date:   2025-03-28 12:00:00 +0100
banner: /assets/img/2025-03/the_futur_of_programming.png
categories: devlog
---

On entend beaucoup parler de Rust ces derniers temps, notamment grâce aux controverses sur son intégration dans le noyau Linux.
C'est un langage qui ne laisse personne indifférent, et pour cause : il a cette fâcheuse tendance à diviser la communauté en deux camps.
D’un côté, ceux qui l’adorent et qui vous expliqueront (avec passion, voire agressivité) à quel point il résout tous les problèmes du monde.
De l’autre, ceux qui, après avoir passé trois heures à se battre contre le borrow checker, se demandent si le C n’était pas une meilleure idée finalement.

![Les devs C quand on leur dit qu'il faut passer au Rust](/assets/img/2025-03/kaamelott-leodagan.gif)
*Les devs C quand on leur dit qu'il faut passer au Rust*

Rust s’est présenté comme le remplaçant du C++, promettant une sécurité mémoire absolue. Plus jamais de segfaults, plus jamais d’UB (undefined behavior), juste du code sûr et performant ! Enfin... sauf si vous essayez de comprendre pourquoi le compilateur refuse obstinément votre code valide sous prétexte que « ça pourrait être dangereux » dans un univers parallèle.

Mais là où Rust est vraiment surprenant, c’est dans son adoption.
Étrangement, une grande partie des développeurs qui l’embrassent viennent d’univers haut niveau comme Python ou JavaScript.
Pourquoi ? Peut-être parce que Rust, avec son package manager intégré et sa philosophie du « fais confiance au compilateur, pas à toi-même », ressemble plus à un langage moderne de haut niveau déguisé en langage système.
Pendant ce temps, les vétérans du C et du C++ haussent un sourcil en voyant des concepts exotiques comme les lifetimes ou la sémantique de l’emprunt qui transforment la programmation en une partie d’échecs contre le compilateur.

Et c’est bien là le problème. En voulant réinventer la roue, Rust a aussi réinventé les routes, les panneaux de signalisation et le code de la route. Résultat ? Un langage puissant, certes, mais qui demande un manuel d’utilisation de 500 pages et une thérapie.

![Les devs Rust en un mot](/assets/img/2025-03/rust-devs-in-a-nutshell.webp)

Mais tout espoir n’est pas perdu. Il existe un langage qui m’a récemment tapé dans l’œil. Il est aussi simple à lire et comprendre que Lua, tout en étant un véritable langage système. Il ne cherche pas à vous imposer un paradigme mentalement épuisant et, surtout, il ne vous fait pas sentir incompétent à chaque erreur de compilation.

J’ose l’affirmer : ce langage est un sérieux candidat pour remplacer le C.

Je parle de Zig.

## Zig

Zig est un langage fabuleux, et ce, pour une raison simple : il ne cherche pas à réinventer toute la syntaxe du monde.
Son objectif est clair : copier le C, mais sans ses absurdités, et en y ajoutant des fonctionnalités modernes et vraiment utiles.

Le résultat ? Un langage qui offre sécurité et performances, tout en restant incroyablement agréable à lire et à écrire. Que demander de plus ?

Cerise sur le gâteau, Zig s'interface parfaitement avec le C, et d’une manière si simple qu’on en viendrait presque à pleurer de joie.
Vous pouvez importer directement des bibliothèques C, sans wrapper absurde, sans générateur de bindings alambiqué.

Et ce n’est pas tout : Zig embarque son propre package manager, Zon, ainsi qu’un système de build écrit... en Zig !
Adieu Make, CMake et autres abominations, bonjour à un système qui ne vous donne pas envie de jeter votre PC par la fenêtre.
(En vrai, il y a un petit temps d'adaptation quand même 😫).
## La gestion mémoire 3.0

Regardons un "Hello, World!" en Zig :

{% highlight zig %}
const std = @import("std");

pub fn main() void {
    std.debug.print("Hello, world!\n", .{});
}
{% endhighlight %}

Jusque-là, rien de perturbant, tout le monde suit. Maintenant, parlons d’un sujet plus sérieux : les allocations mémoire.

En Zig, aucune allocation n’est cachée. Vous savez toujours où, quand et comment la mémoire est allouée et libérée.
Un concept révolutionnaire pour certains langages modernes, apparemment.

Examinons un exemple avec un allocateur de type "Arena" :

```zig
const std = @import("std");

pub fn main() void {
    var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
    defer arena.deinit();
    const allocator = arena.allocator();

    _ = try allocator.alloc(u8, 1);
    _ = try allocator.alloc(u8, 10);
    _ = try allocator.alloc(u8, 100);
}
```

À ce stade, vous avez peut-être quelques questions. Commençons par les plus évidentes :

#### Pourquoi ces _ = devant les allocations ?

Parce que Zig est un langage intelligent. Il vous avertit si vous déclarez une variable sans l’utiliser.
Si vous voulez ignorer la valeur retournée sans faire hurler le compilateur, utilisez _.

#### Que fait std.heap.ArenaAllocator ?

C’est un allocateur secondaire qui s’appuie sur un allocateur primaire (ici, std.heap.page_allocator).
Autrement dit, il vous permet de mieux gérer vos allocations en fonction de vos besoins.

#### Pourquoi tout ce charabia alors que malloc() existe ?

Parce que le vraie GOAT, c’est `defer`.

### Le mot-clé defer, ou comment dire adieu aux fuites mémoire

Le mot-clé `defer`, c’est un petit bijou.
Il permet d’exécuter une instruction automatiquement à la sortie du bloc.

Autrement dit, vous pouvez allouer de la mémoire et être certain qu’elle sera libérée.
Finies les fuites mémoire accidentelles, les `free()` oubliés, et les cauchemars post-traumatiques liés à RAII en C++.

Vous allouez, et vous libérez juste après avec un simple `defer`.

C’est quand même plus simple que Rust, non ?

### Et la cerise sur le gâteau ?

La deuxième surprise, c’est ce qui vient après le `defer`.

Toute la mémoire allouée avec notre allocateur mémoire est libérée d’un coup lorsqu’on libère l’allocateur lui-même.

Autrement dit, pas besoin de traquer chaque allocation individuellement : il suffit de libérer l’allocateur pour que tout disparaisse proprement.
Quelle simplicité !

### L’allocateur fixe : allocation instantanée, zéro overhead

Allouer de la mémoire, c’est coûteux. Ce serait bien de tout allouer d’un coup, puis de distribuer les blocs selon nos besoins.
Eh bien, Zig permet de faire exactement ça :

```zig
const std = @import("std");

pub fn main() void {
    var buffer: [1000]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    const memory = try allocator.alloc(u8, 100);
    defer allocator.free(memory);
}
```

Ici, on utilise un buffer pré-alloué et un allocateur fixe (FixedBufferAllocator).
Résultat : aucun appel système inutile, aucune fragmentation mémoire, et des performances au top.

## Conclusion

Le but de cet article n’était pas d’être exhaustif, mais de vous montrer une chose essentielle : la gestion mémoire n’a pas besoin d’être une corvée.

En Rust, on vous impose un borrow checker parfois aussi coopératif qu’un huissier de justice.
En C++, vous jonglez avec des smart pointers et des segfaults inopinés.
En Zig ? Vous avez le contrôle, la simplicité, et des outils efficaces.

Je vous encourage vivement à explorer Zig par vous-même.
Ce langage est d’une clarté remarquable, et contrairement à Rust, on ne se sent pas complètement perdu en l’utilisant.

J’ose même dire que Zig pourrait devenir un candidat sérieux pour le noyau Linux.
À condition que ce dernier ne continue pas de s’enfoncer dans le nouveau Java qu’est Rust. 
