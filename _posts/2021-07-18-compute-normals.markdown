---
layout: post
title:  "Calculer les normales pour l'éclairage"
date:   2021-07-18 18:48:00 +0100
categories: devlog
---

Récemment, je me suis penché sur la question de l'éclairage pour mon moteur de rendu. Qui dit éclairage, dit forcément normales. Calculer des normales en soi n'est pas compliqués, mais selon la méthode de calcul, on se retrouve avec des résultats très différents. Dans un monde parfait, les normales sont fournies par l'artiste à l'origine du modèle 3D. Après tout, il est le seul à savoir comment sa création réagit à la lumière. En revanche, dans le cas d'une création procédurale, c'est un peu plus compliqué.

## Calculer la normale d'un sommet

Pour réaliser un bon éclairage, la base c'est de calculer la normale de chaque triangle et de l'assigné au sommet concerner.  
La formule pour cela est très simple : $$N = (B - A) \times (C - A)$$ avec $$N$$ la normale et $$A, B, C$$ les sommets.

Rien de bien compliqué. La subtilité, c'est qu'un sommet peut être réutilisé dans plusieurs triangles. Un sommet aurait donc plusieurs normales ?

## Smooth Shading vs Hard Shading

Pour répondre à cette question, il faut d'abord se demander quel résultat on souhaite obtenir, mais aussi le type d'objet avec lequel on travaille.

Dans le cas d'un cube, les faces sont perpendiculaires aux faces adjacentes. Autrement dit, l'éclairage d'une face à une autre change radicalement. C'est ce qu'on va appeler du hard shading.

<iframe style="margin:1rem auto; display: block;" width="700" height="394" src="https://www.youtube.com/embed/abdZcInc1OI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Pour ce faire, il n'y a pas trente-six solutions. Il faut calculer une normale pour chaque face et l'attribuer au sommet de cette face. Par exemple dans le cas, d'un cube, on se retrouve avec trois normales par sommet.

Dans le cas d'une sphère, on a l'opposé. Une sphère est lisse donc l'éclairage à sa surface est gradué. Si l’on utilise du hard shading, on remarquerait que les faces qui composent la sphère seraient visibles et bien distinctes. On veut l'exact opposé ! Pour cela, un smooth shading va être utilisé. La méthode consiste à faire pour chaque sommet, la somme des normales de ses faces. Somme que l'on normalise. Ainsi on obtient une moyenne qui nous permet d'effectuer de joli dégradé.

<iframe style="margin:1rem auto; display: block;" width="700" height="394" src="https://www.youtube.com/embed/ZRR2yRtXMkU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Ma solution pratique

Même si le smooth shading et le hard shading sont différent, en réalité, il en retourne de la même fonction : calculer une normale. La différence réside surtout dans le fait que pour l'un on fait une somme et pour l'autre on garde des valeurs multiples bien distinctes. L'un des problèmes que l'on rencontre avec cette dernière méthode, c'est qu'on ne puisse pas associer la bonne normale à la bonne face depuis le vertex ou le fragment shader. Ce qui signifie qu'on est obligé de dupliquer les sommets. Mais cela nous arrange bien, car on va pouvoir combiner le smooth shading et le hard shading.

Il nous faut d'abord une structure qui détient nos indices et nos sommets.

    struct vertex
    {
        glm::vec3 pos;
	    glm::vec3 normal;
    }

    struct Geometry
    {
        std::vector<vertex> vertices; // les sommets
        std::vector<int32_t> indices; // les indices
    }

Chaque sommet va avoir un indice qui lui est attribué. On peut créer une fonction qui a pour objectif d'ajouter un sommet à notre objet et de retourner son indice.

    int32_t addVertex(const vertex& vertex)
    {
        // vertex.normal doit être { 0.0f, 0.0f, 0.0f }
        vertices.push(vertex);

        /* le moins -1 n'est pas obligatoire mais,
        je prefere que le premier indice soit 0 pour 
        mapper l'indices à l'index dans le vector */
        return vertices.size() - 1;
    }

Une fois que tous les sommets ont été créés, on peut commencer à former les triangles de l'objet en ajoutant nos indices trois par trois. Dans le même temps, on peut calculer la normale du triangle puisque chaque indice correspond à un sommet.

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

		// On ajoute notre normale a celle de chaque sommet du triangle
		vertices[x].normal += glm::normalize(vertices[x].normal + normal);
		vertices[y].normal += glm::normalize(vertices[y].normal + normal);
		vertices[z].normal += glm::normalize(vertices[z].normal + normal);

        // On ajoute les indices a notre liste
        indices.push(x);
        indices.push(y);
        indices.push(z);
    }

Le code fonctionne de manière très simple : si on veut du smooth shading, il suffit de réutilisé les mêmes sommets sur plusieurs faces adjacentes. La normale qui sera calculée avec cet indice sera ajoutée à la normale du sommet correspondante.

    // CREATION D'UN CUBE
	const float half_size = size / 2;

    Geometry geo; // Notre mesh

	// On défini nos sommets
	uint32_t a = geo.addVertex({ -half_size, -half_size, -half_size }, vcolor, { .0f, .0f });
	uint32_t b = geo.addVertex({ half_size, -half_size, -half_size }, vcolor, { .0f, 2.f });
	uint32_t c = geo.addVertex({ -half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t d = geo.addVertex({ half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t e = geo.addVertex({ -half_size, -half_size, half_size }, vcolor, { .0f, .0f });
	uint32_t f = geo.addVertex({ half_size, -half_size, half_size }, vcolor, { .0f, 2.f });
	uint32_t g = geo.addVertex({ -half_size, half_size, half_size }, vcolor, { 2.f, .0f });
	uint32_t h = geo.addVertex({ half_size, half_size, half_size }, vcolor, { 2.f, 2.f });

	// On reutilise les sommets pour construire toute les faces du cube 
	geo.addIndices(a, c, b);
	geo.addIndices(b, c, d);

	geo.addIndices(f, h, e);
	geo.addIndices(e, h, g);

	geo.addIndices(b, d, f);
	geo.addIndices(f, d, h);

	geo.addIndices(e, g, a);
	geo.addIndices(a, g, c);

	geo.addIndices(c, g, d);
	geo.addIndices(d, g, h);

	geo.addIndices(e, a, f);
	geo.addIndices(f, a, b);

Dans le cas d'un hard shading, on va simplement utiliser des indices différents. Ainsi, les normales des différentes faces ne seront pas sommées.

    // CREATION D'UN CUBE
	const float half_size = size / 2;

    Geometry geo; // Notre mesh

	// Chaque sommets est partagé par trois faces. Il faut donc les définir trois fois.
	uint32_t a = geo.addVertex({ -half_size, -half_size, -half_size }, vcolor, { .0f, .0f });
	uint32_t b = geo.addVertex({ half_size, -half_size, -half_size }, vcolor, { .0f, 2.f });
	uint32_t c = geo.addVertex({ -half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t d = geo.addVertex({ half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t e = geo.addVertex({ -half_size, -half_size, half_size }, vcolor, { .0f, .0f });
	uint32_t f = geo.addVertex({ half_size, -half_size, half_size }, vcolor, { .0f, 2.f });
	uint32_t g = geo.addVertex({ -half_size, half_size, half_size }, vcolor, { 2.f, .0f });
	uint32_t h = geo.addVertex({ half_size, half_size, half_size }, vcolor, { 2.f, 2.f });

	uint32_t a1 = geo.addVertex({ -half_size, -half_size, -half_size }, vcolor, { .0f, .0f });
	uint32_t b1 = geo.addVertex({ half_size, -half_size, -half_size }, vcolor, { .0f, 2.f });
	uint32_t c1 = geo.addVertex({ -half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t d1 = geo.addVertex({ half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t e1 = geo.addVertex({ -half_size, -half_size, half_size }, vcolor, { .0f, .0f });
	uint32_t f1 = geo.addVertex({ half_size, -half_size, half_size }, vcolor, { .0f, 2.f });
	uint32_t g1 = geo.addVertex({ -half_size, half_size, half_size }, vcolor, { 2.f, .0f });
	uint32_t h1 = geo.addVertex({ half_size, half_size, half_size }, vcolor, { 2.f, 2.f });

	uint32_t a2 = geo.addVertex({ -half_size, -half_size, -half_size }, vcolor, { .0f, .0f });
	uint32_t b2 = geo.addVertex({ half_size, -half_size, -half_size }, vcolor, { .0f, 2.f });
	uint32_t c2 = geo.addVertex({ -half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t d2 = geo.addVertex({ half_size, half_size, -half_size }, vcolor, { 2.f, .0f });
	uint32_t e2 = geo.addVertex({ -half_size, -half_size, half_size }, vcolor, { .0f, .0f });
	uint32_t f2 = geo.addVertex({ half_size, -half_size, half_size }, vcolor, { .0f, 2.f });
	uint32_t g2 = geo.addVertex({ -half_size, half_size, half_size }, vcolor, { 2.f, .0f });
	uint32_t h2 = geo.addVertex({ half_size, half_size, half_size }, vcolor, { 2.f, 2.f });

	// On ne dois pas reutiliser les sommets pour chaque face
	geo.addIndices(a, c, b);
	geo.addIndices(b, c, d);

	geo.addIndices(f, h, e);
	geo.addIndices(e, h, g);

	geo.addIndices(b1, d1, f1);
	geo.addIndices(f1, d1, h1);

	geo.addIndices(e1, g1, a1);
	geo.addIndices(a1, g1, c1);

	geo.addIndices(c2, g2, d2);
	geo.addIndices(d2, g2, h2);

	geo.addIndices(e2, a2, f2);
	geo.addIndices(f2, a2, b2);

## Conclusion

J'espère vous avoir appris de nouvelles choses. Je ferais probablement d'autres posts sur l'éclairage, car il y a très peu de ressources en français et encore moins qui sont spécifiques à une implémentation avec Vulkan. Je vous mets en lien mon moteur de rendu pour que vous puissiez avoir un exemple d'application, ainsi que quelques sources qui m'ont aidé à naviguer vers cette solution. Sur ce, portez-vous bien et à la prochaine.

[Code source R3DEngine](https://github.com/MrScriptX/R3D_Engine)  
[Introduction to Shading](https://www.scratchapixel.com/lessons/3d-basic-rendering/introduction-to-shading/shading-normals)  
[Basic Lighting](https://learnopengl.com/Lighting/Basic-Lighting)  
[Calculating normals in a triangle mesh](https://stackoverflow.com/questions/6656358/calculating-normals-in-a-triangle-mesh/6661242#6661242)  
