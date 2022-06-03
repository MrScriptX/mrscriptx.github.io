---
layout: post
title:  "Integrer Dear ImGui dans un moteur de rendu"
date:   2022-06-03 12:00:00 +0100
categories: devlog
---

Dernièrement, j'étais coincé sur un problème de fuite de mémoire. Le cauchemar de tous les développeurs C++. Je me suis vite rendu compte que quand même, ça serait plus simple si j'avais une interface graphique dans mon jeu. Le souci, c'est qu'intégrer une interface, c'est beaucoup de travail. Je devrais démarrer un chantier juste pour trouver cette fuite de mémoire. Non ! Il y a forcement plus simple. Ce plus simple, c'est Dear ImGui.

Dear ImGui est une bibliothèque très légère et super facile à utiliser permettant d'intégrer une interface à n'importe quel moteur utilisant n'importe quel API. Après quelques galères pour déchiffrer la documentation, j'ai finalement réussi à l'implémenter.

## Les objets Vulkan

Le plus simple pour intégrer ImGui a son moteur de rendu, c'est de lui créer un pipeline propre.
Ce qui signifie qu'il lui faut des objets Vulkan propres.
À savoir :
- Un VkRenderPass
- Un VkCommandPool avec lequel on va allouer des commands buffers pour ImGui seulement
- Des VkCommandBuffer. Dans mon cas, j'en ai 3, une pour chaque image de la swapchain.
- Des VkFramebuffer. Tout comme les commands buffers, un par image de la swapchain.
- Un VkDescriptorPool qui est très optionnel puisqu'on pourrait étendre le descriptor pool qui existe déjà.

Ce qui nous donne au final ceci.

```
struct UIVulkanObject
{
    VkCommandPool command_pool;
    std::vector<VkCommandBuffer> command_buffers;
    std::vector<VkFramebuffer> framebuffers;
    VkRenderPass render_pass;
    VkDescriptorPool decriptor_pool = VK_NULL_HANDLE;
}
```

Bon, vous allez me dire, tu es bien gentil avec ta structure. Mais on initialise ça où ? Ne vous inquiétez pas, c'est très simple. 
Initialiser ces objets en même temps que ceux d'origine.
Quand vous créez votre command pool de base, créer celle pour l'UI juste après, pareil pour le render pass, etc.
La subtilité c'est qu'il y a un peu de travail supplémentaire concernant l'initialisation, mais il y a quand même quelques différences à noter.

Le descriptor pool va être bien plus grand.

```
std::array<VkDescriptorPoolSize, 11> pool_sizes = {};
pool_sizes[0].type = VK_DESCRIPTOR_TYPE_SAMPLER;
pool_sizes[0].descriptorCount = 1000;
pool_sizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
pool_sizes[1].descriptorCount = 1000;
pool_sizes[2].type = VK_DESCRIPTOR_TYPE_SAMPLED_IMAGE;
pool_sizes[2].descriptorCount = 1000;
pool_sizes[3].type = VK_DESCRIPTOR_TYPE_STORAGE_IMAGE;
pool_sizes[3].descriptorCount = 1000;
pool_sizes[4].type = VK_DESCRIPTOR_TYPE_UNIFORM_TEXEL_BUFFER;
pool_sizes[4].descriptorCount = 1000;
pool_sizes[5].type = VK_DESCRIPTOR_TYPE_STORAGE_TEXEL_BUFFER;
pool_sizes[5].descriptorCount = 1000;
pool_sizes[6].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
pool_sizes[6].descriptorCount = 1000;
pool_sizes[7].type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
pool_sizes[7].descriptorCount = 1000;
pool_sizes[8].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC;
pool_sizes[8].descriptorCount = 1000;
pool_sizes[9].type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER_DYNAMIC;
pool_sizes[9].descriptorCount = 1000;
pool_sizes[10].type = VK_DESCRIPTOR_TYPE_INPUT_ATTACHMENT;
pool_sizes[10].descriptorCount = 1000;

VkDescriptorPoolCreateInfo pool_info = {};
pool_info.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
pool_info.poolSizeCount = static_cast<uint32_t>(pool_sizes.size());
pool_info.pPoolSizes = pool_sizes.data();
pool_info.maxSets = 100;

if (vkCreateDescriptorPool(m_graphic.device, &pool_info, nullptr, &m_ui.decriptor_pool) != VK_SUCCESS)
{
	throw std::runtime_error("failed to create descriptor pool!");
}
```

Le render pass va changer également, car c'est celui d'ImGui qui va être présenté en dernier. 
Donc il faut mettre à jour tout les render pass en conséquence. 
À savoir, il ne faut pas oublier de mettre le color attachment du render pass de notre moteur à VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL et non plus VK_IMAGE_LAYOUT_PRESENT_SRC_KHR.

```
// render pass ImGui
VkAttachmentDescription attachment = {};
attachment.format = m_graphic.swapchain_details.format;
attachment.samples = VK_SAMPLE_COUNT_1_BIT;
attachment.loadOp = VK_ATTACHMENT_LOAD_OP_LOAD;
attachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
attachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
attachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachment.initialLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
attachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR; // on indique que c'est notre dernier render pass

VkAttachmentReference color_attachment = {};
color_attachment.attachment = 0;
color_attachment.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &color_attachment;

VkSubpassDependency dependency = {};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;

VkRenderPassCreateInfo info = {};
info.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
info.attachmentCount = 1;
info.pAttachments = &attachment;
info.subpassCount = 1;
info.pSubpasses = &subpass;
info.dependencyCount = 1;
info.pDependencies = &dependency;
if (vkCreateRenderPass(m_graphic.device, &info, nullptr, &m_ui.render_pass) != VK_SUCCESS)
{
	throw std::runtime_error("Could not create Dear ImGui's render pass");
}
```

## Initialisation d'ImGui

Après avoir créé tous vos objets, il faut initialiser ImGui.

```
// create render pass, command pool, allocate command buffer, etc...

IMGUI_CHECKVERSION();
ImGui::CreateContext();
ImGuiIO& io = ImGui::GetIO();
(void)io;
```

Ensuite il faut fournir à ImGui les objets Vulkan dont il aura besoin.

```
ImGui_ImplGlfw_InitForVulkan(&window, true); // handle à la GLFWwindow déjà instancié
ImGui_ImplVulkan_InitInfo init_info = {};
init_info.Instance = m_graphic.instance;
init_info.PhysicalDevice = m_graphic.physical_device;
init_info.Device = m_graphic.device;
init_info.QueueFamily = m_graphic.queue_indices.graphic_family;
init_info.Queue = m_graphic.graphics_queue;
init_info.PipelineCache = VK_NULL_HANDLE;
init_info.DescriptorPool = m_ui.decriptor_pool;
init_info.Allocator = nullptr;
init_info.MinImageCount = m_graphic.swapchain_images.size();
init_info.ImageCount = m_graphic.framebuffers.size();
init_info.CheckVkResultFn = nullptr;
ImGui_ImplVulkan_Init(&init_info, m_ui.render_pass);
```

Pour finir, il faut uploader les textures vers le GPU. Pour cela, on va créer un command buffer, puis le désallouer dans la fouler. Pas besoin de faire un objet global pour cela, il ne servira plus.

```
VkCommandBuffer command_buffer = beginCommands();
ImGui_ImplVulkan_CreateFontsTexture(command_buffer);
endCommands(command_buffer);
ImGui_ImplVulkan_DestroyFontUploadObjects();
```

## Le rendu

Voilà nous avons fini la préparation, maintenant, nous allons enregistrer les commandes dans les commands buffers dédiés à ImGui. 
Pour cela, il suffit de faire une fonction que nous appellerons dans notre update juste après avoir enregistré nos commands buffers pour le rendu classique.

```
void Renderer::UpdateUI()
{
	VkCommandBufferBeginInfo cmdBufferBegin = {};
	cmdBufferBegin.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
	cmdBufferBegin.flags |= VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

	if (vkBeginCommandBuffer(m_ui.command_buffers[m_current_image], &cmdBufferBegin) != VK_SUCCESS)
	{
		throw std::runtime_error("Unable to start recording UI command buffer!");
	}

	VkClearValue clearColor = { 0.0f, 0.0f, 0.0f, 1.0f };
	VkRenderPassBeginInfo renderPassBeginInfo = {};
	renderPassBeginInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
	renderPassBeginInfo.renderPass = m_ui.render_pass;
	renderPassBeginInfo.framebuffer = m_ui.framebuffers[m_current_image];
	renderPassBeginInfo.renderArea.extent.width = m_graphic.swapchain_details.extent.width;
	renderPassBeginInfo.renderArea.extent.height = m_graphic.swapchain_details.extent.height;
	renderPassBeginInfo.clearValueCount = 1;
	renderPassBeginInfo.pClearValues = &clearColor;

	vkCmdBeginRenderPass(m_ui.command_buffers[m_current_image], &renderPassBeginInfo, VK_SUBPASS_CONTENTS_INLINE);

	// Grab and record the draw data for Dear Imgui
	ImGui_ImplVulkan_RenderDrawData(ImGui::GetDrawData(), m_ui.command_buffers[m_current_image]);

	// End and submit render pass
	vkCmdEndRenderPass(m_ui.command_buffers[m_current_image]);

	if (vkEndCommandBuffer(m_ui.command_buffers[m_current_image]) != VK_SUCCESS)
	{
		throw std::runtime_error("Failed to record command buffers!");
	}
}
```

Plus qu'à afficher en mettant à jour notre VkSubmitInfo.

```
// on passe les deux commands buffers ensemble
std::array<VkCommandBuffer, 2> command_buffers = { m_graphic.command_buffers[m_current_image], m_ui.command_buffers[m_current_image] };
submit_info.commandBufferCount = command_buffers.size();
submit_info.pCommandBuffers = command_buffers.data();
```

Ensuite on peut dessiner en appelant les fonctions d'ImGui. Ce qui nous donne le workflow final suivant :

```
// on dessine une fenetre
ImGui_ImplVulkan_NewFrame();
ImGui_ImplGlfw_NewFrame();
ImGui::NewFrame();

ImGui::Begin("Ma fenetre");
ImGui::End();

ImGui::Render();

mp_renderer->UpdateGame(); // on enregistre les commands buffers du jeu
mp_renderer->UpdateUI(); // on enregistre les commands buffers ImGui
mp_renderer->draw(); // on submit le tout à la VkQueue
```

## Les inputs

Vous pouvez lancer votre programme, normalement, une petite fenêtre devrait s'afficher par-dessus votre rendu. 
Mais, il y a un défaut, c'est que si vous avez votre souris libre, vous remarquerez que vous ne pouvez pas interagir avec cette fenêtre. 
La raison est simple. ImGui ne sait pas où se trouve votre souris, il faut lui indiquer.

Dans votre callback qui gère votre souris. Passer la position de votre souris aux fonctions d'ImGui.

```
void Window::mouse_callback(GLFWwindow* window, double xpos, double ypos)
{
	ImGuiIO& io = ImGui::GetIO();
	io.MousePos = ImVec2(xpos, ypos);
}
```

Et voilà, tout est prêt !

## Conclusion

ImGui est une bibliothèque très intéressante et facile d'utilisation qui m'a permis de régler mon problème de fuite mémoire (même si j'en ai une autre fuite du coup).
Je ferais un autre article sur mon implémentation dans mon moteur.
Vous pouvez toujours retrouver le code vers le GitHub du projet afin de vous aider (sur la branche develop).

Sur ce, je vous souhaite un bon voyage.

[r3d-engine]: https://github.com/MrScriptX/R3D_Engine