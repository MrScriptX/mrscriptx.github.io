---
layout: post
title:  "Calculer les normales pour l'éclairage"
date:   2021-06-12 22:32:00 +0100
categories: devlog
---

Récemment, je me suis pencher sur la question de l'éclairage pour mon moteur de rendu. Qui dit éclairage, dit forcément normales. Calculer des normales en soit n'est pas compliqués en soit, mais selon la méthode de calcul, on se retrouve avec des resultats bien très différents. Dans un monde parfait, les normales sont fournis par l'artiste qui crée le modèle 3D. Apres tout, il est bien le seul à savoir à quoi doit ressembler son modèle. En revanche, pour du procédural, c'est un peu plus compliqué.

## Calculer la normale d'un vertex

Pour réaliser un bonne éclairage, la base c'est de calculer la normale de chaque triangle et de l'assigné au vertex concerner. La formule pour cela est très simple.
N = (B - A) x (C - A)
Avec N la normale et A, B, C les vertex.

Rien de bien compliqué. La subtilité, c'est qu'un vertex peut-etre réutiliser dans plusieurs triangles. Un vertex aurait donc plusieurs normales ?

## Smooth Shading vs Hard Shading

Pour répondre à cette question, il faut d'abord se demander qu'elle resultat on souhaite obtenir, mais aussi le type d'objet avec lequel on travaille.

Dans le cas d'un cube, les faces sont perpendiculaires à celles ajdcentes. Autrement dit, l'éclairage d'une face à une autre change radicalement. C'est ce qu'on va appelé du hard shading

<iframe style="margin:0 auto; display: block;" width="700" height="394" src="https://www.youtube.com/embed/abdZcInc1OI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Pour ce faire, il n'y pas trente six solutions. Il faut calculer une normale pour chaque face est l'attribué aux vertex de cette face. Donc dans le cas, d'un cube, on trois normales par vertex.

Dans le cas d'une sphere, on a l'opposé. Une sphere est lisse donc l'eclairage à sa surface est gradué. SI on utilise du hard shading, on remaquerai que les faces qui composent la sphere serait visible et bien distincte. On veux l'opposé ! Pour cela un smooth shading va être utilisé. La méthode consiste à faire pour chaque vertex, la somme des normales de ses faces. Somme qu'il faudra normalisé. Ainsi on obtient une moyenne est on peux effectué de jolie degradé.

<iframe style="margin:0 auto; display: block;" width="700" height="394" src="https://www.youtube.com/embed/ZRR2yRtXMkU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Ma solution pratique

Même si le smooth shading et le hard shading sont different, en réalité, il en retourne du même calcul. Calculer une normale. La difference reside surtout dans le fait que pour l'un on fait une somme et pour l'autre on garde des valeurs multiples bien distinctes. L'un des problemes que l'on rencontre avce cette derniere methode, c'est qu'on ne sait pas comment associer la bonne normale à la bonne face depuis le vertex ou le fragment shader. Ce qui signifie qu'on est obliger de dupliquer les vertex. Mais cela nous arrange bien, car on va pouvoir combiner le smooth shading et le hard shading.

Il nous faut d'abord une structure qui detiennent nos indices et nos vertex.

    struct Vertex
    {
        glm::vec3 pos;
	    glm::vec3 normal;
    }

    struct Geometry
    {
        std::vector<Vertex> vertices; // les vertex
        std::vector<int32_t> indices; // les indices
    }

Chaque vertex va avoir un indice qui lui est attribué. On peut creer un fonction qui permet de d'ajouter un vertex et de retourner son indice.

    int32_t addVertex(const Vertex& vertex)
    {
        // vertex.normal doit être { 0.0f, 0.0f, 0.0f }
        vertices.push(vertex);

        /* le moins -1 n'est pas obligatoire mais,
        je prefere que le premier indice soit 0 pour 
        mapper l'indices à l'index dans le vector */
        return vertices.size() - 1;
    }

Une fois que toute les vertex ont été créé, on peux commencer a ajouter les triangles en ajoutant nos indices trois par trois. Dans le même temps, on peux calculer la normale du triangle puisque chaque indice correspond à un vertex.

    // x, y, z ne represente pas des coordonnées. Ce sont des noms arbitraires
    void addTriangle(int32_t x, int32_t y, int32_t z)
    {
        // on calcule la normale du triangle
        glm::vec3 a = {
			vertices[x].pos.x - vertices[y].pos.x,
			vertices[x].pos.y - vertices[y].pos.y,
			vertices[x].pos.z - vertices[y].pos.z,
		};

		glm::vec3 b = {
			vertices[z].pos.x - vertices[y].pos.x,
			vertices[z].pos.y - vertices[y].pos.y,
			vertices[z].pos.z - vertices[y].pos.z,
		};

		glm::vec3 normal = glm::cross(a, b);

		// On ajoute notre normale a celle de chaque vertex du triangle
		vertices[x].normal += glm::normalize(vertices[x].normal + normal);
		vertices[y].normal += glm::normalize(vertices[y].normal + normal);
		vertices[z].normal += glm::normalize(vertices[z].normal + normal);

        // On ajoute les indices a notre liste
        indices.push(x);
        indices.push(y);
        indices.push(z);
    }

Le code fonctionne de maniere tres simple : si on veut du smooth shading, il suffit de réutilisé le même indice (ou vertex). La normale qui sera calculé avec cet indice sera ajouter à la normale de la vertex correspondante.

Dans le cas d'un hard shading, on va simplement utiliser des indices differents. Ainsi, les normales des differentes faces ne seront pas sommer.

## Conclusion

J'espere vous avoir de nouvelles choses. Je ferais probablement d'autres posts sur l'éclairage car il y a très peu de ressource en francais et encore moins spécifique à une implementation avec Vulkan. Je vous mets en lien mon moteur de rendu pour que vous puissiez avoir un exemple d'application, ainsi que quelques sources qui m'ont aider à naviguer vers cette solution. Sur ce, portez vous bien et à la prochaine.

[Moteur R3DEngine](https://github.com/MrScriptX/R3D_Engine)
