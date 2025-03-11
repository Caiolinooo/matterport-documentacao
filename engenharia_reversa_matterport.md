# Engenharia Reversa do Matterport através do matterport-dl

## 1. Introdução à Ferramenta matterport-dl

O matterport-dl é uma ferramenta de código aberto que permite baixar, arquivar e visualizar localmente os tours virtuais do Matterport. Esta ferramenta é extremamente valiosa para entender a estrutura interna do sistema Matterport, pois precisa replicar todo o funcionamento do sistema de visualização original.

A análise dessa ferramenta nos permite compreender:
- Como os dados Matterport são estruturados
- Como o sistema de visualização funciona
- Como os diferentes componentes interagem
- Quais APIs e formatos de dados são utilizados

## 2. Arquitetura de Arquivos e Estrutura de Dados

Ao examinar o comportamento do matterport-dl, identificamos a seguinte estrutura de arquivos em um tour Matterport típico:

```
downloads/
  └── {model_id}/
      ├── assets/
      │   ├── js/            (Scripts JavaScript do viewer)
      │   ├── css/           (Estilos da interface)
      │   └── images/        (Ícones e recursos gráficos)
      ├── files/
      │   ├── showcase.json  (Metadados principais do modelo)
      │   ├── skybox/        (Texturas de céu para ambientes externos)
      │   ├── sweep/         (Dados de panoramas)
      │   │   ├── {sweep_id}/
      │   │   │   ├── high_res.jpg (Panorama em alta resolução)
      │   │   │   ├── medium_res.jpg (Panorama em média resolução)
      │   │   │   ├── low_res.jpg (Panorama em baixa resolução)
      │   │   │   └── depth.png   (Mapa de profundidade)
      │   │   └── ...
      │   ├── mesh/          (Dados da malha 3D)
      │   │   ├── model.obj  (Geometria da malha)
      │   │   ├── model.mtl  (Materiais da malha)
      │   │   └── textures/  (Texturas da malha)
      │   ├── floorplan/     (Dados de planta baixa)
      │   └── mattertags/    (Dados de anotações)
      ├── index.html         (Página principal)
      └── run_args.json      (Configurações do download)
```

### 2.1 Formato de Arquivos Chave

#### showcase.json
Este arquivo contém metadados fundamentais sobre o modelo, incluindo:

```json
{
  "model": {
    "id": "{model_id}",
    "name": "Nome do Modelo",
    "version": "versão",
    "sweeps": [
      {
        "id": "{sweep_id}",
        "position": [x, y, z],
        "rotation": [qx, qy, qz, qw],
        "neighbors": ["{sweep_id_vizinho}", ...]
      },
      ...
    ],
    "floorCount": 2,
    "floors": [
      {
        "id": "{floor_id}",
        "name": "Piso 1",
        "sweeps": ["{sweep_id}", ...]
      },
      ...
    ],
    "mattertags": [
      {
        "id": "{tag_id}",
        "position": [x, y, z],
        "normal": [nx, ny, nz],
        "title": "Título",
        "description": "Descrição",
        "media": {...}
      },
      ...
    ]
  }
}
```

#### Arquivos de Panorama
Os panoramas são armazenados em diferentes resoluções para otimização de carregamento:
- **high_res.jpg**: Panorama completo em alta resolução (tipicamente 8192x4096 pixels)
- **medium_res.jpg**: Versão média (tipicamente 4096x2048 pixels)
- **low_res.jpg**: Versão de baixa resolução para pré-carregamento (tipicamente 2048x1024 pixels)
- **depth.png**: Mapa de profundidade codificado em 16 bits (valores de 0 a 65535)

#### Arquivos de Malha 3D
A malha 3D é armazenada em formatos padrão:
- **model.obj**: Geometria em formato Wavefront OBJ
- **model.mtl**: Definições de materiais
- **textures/**: Texturas em formato JPEG ou PNG

## 3. Sistema de Renderização e Visualização

Analisando o código JavaScript baixado pelo matterport-dl, podemos identificar como o Matterport implementa seu sistema de visualização:

### 3.1 Inicialização do Viewer

O processo de inicialização envolve:
1. Carregamento dos metadados do modelo (showcase.json)
2. Configuração do contexto WebGL
3. Inicialização da cena Three.js
4. Carregamento do panorama inicial e recursos necessários
5. Configuração dos controles de câmera e interação

```javascript
// Pseudocódigo baseado na análise do código real
function initViewer(modelId) {
  const modelData = await fetchModelData(modelId);
  const container = document.getElementById('showcase-container');
  
  // Inicializar Three.js
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  
  // Configurar controles de câmera
  const controls = new THREE.DeviceOrientationControls(camera);
  
  // Inicializar camadas de visualização
  initPanoramaLayer(scene, modelData);
  initModelLayer(scene, modelData);
  initUILayer(scene, modelData);
  
  // Configurar eventos e loop de renderização
  setupEventHandlers(container, camera, controls);
  startRenderLoop(renderer, scene, camera, controls);
}
```

### 3.2 Sistema de Navegação

O sistema de navegação é baseado em um grafo de conectividade entre panoramas:

```javascript
// Pseudocódigo do sistema de navegação
function navigateTo(targetSweepId) {
  const currentSweep = getCurrentSweep();
  const targetSweep = getSweep(targetSweepId);
  
  // Calcular caminho de transição
  const startPosition = currentSweep.position;
  const endPosition = targetSweep.position;
  
  // Iniciar animação de transição
  startTransition({
    duration: 1000, // milissegundos
    onUpdate: (progress) => {
      // Interpolação de posição
      const currentPos = lerpVector(startPosition, endPosition, progress);
      camera.position.copy(currentPos);
      
      // Transição de panoramas
      blendPanoramas(currentSweep.panorama, targetSweep.panorama, progress);
      
      // Atualizar interface
      updateUI(progress);
    },
    onComplete: () => {
      setCurrentSweep(targetSweep);
      loadNeighboringSweeps(targetSweep);
    }
  });
}
```

### 3.3 Renderização de Panoramas

Os panoramas são renderizados em uma esfera ou cubemap ao redor da câmera:

```javascript
// Pseudocódigo da renderização de panorama
function createPanoramaSphere(panoramaTexture) {
  const geometry = new THREE.SphereGeometry(50, 64, 32);
  // Inverter a geometria para renderizar por dentro
  geometry.scale(-1, 1, 1);
  
  const material = new THREE.MeshBasicMaterial({
    map: panoramaTexture,
    side: THREE.BackSide
  });
  
  return new THREE.Mesh(geometry, material);
}

function blendPanoramas(textureA, textureB, blend) {
  // Usar shader personalizado para misturar panoramas
  const blendMaterial = new THREE.ShaderMaterial({
    uniforms: {
      textureA: { value: textureA },
      textureB: { value: textureB },
      blend: { value: blend }
    },
    vertexShader: /* GLSL vertex shader */,
    fragmentShader: /* GLSL fragment shader para blending */
  });
  
  panoramaSphere.material = blendMaterial;
}
```

### 3.4 Renderização da Malha 3D

Para os modos "dollhouse" e planta baixa, o modelo 3D é carregado e renderizado:

```javascript
// Pseudocódigo da renderização de modelo 3D
async function loadModel(modelUrl) {
  const loader = new THREE.OBJLoader();
  const model = await loader.loadAsync(modelUrl);
  
  // Configurar materiais e texturas
  model.traverse(child => {
    if (child instanceof THREE.Mesh) {
      // Aplicar materiais e textura
      applyMaterials(child);
      
      // Configurar sombras
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
  
  return model;
}

function switchToDollhouseView() {
  // Animar transição de câmera
  moveCameraToOverview();
  
  // Mostrar modelo 3D, esconder panorama
  modelGroup.visible = true;
  panoramaGroup.visible = false;
  
  // Configurar iluminação para dollhouse
  setupDollhouseLighting();
}
```

## 4. Análise das APIs e Endpoints

Através da análise do código do matterport-dl, podemos identificar os principais endpoints e APIs que o Matterport utiliza:

### 4.1 APIs REST

```
GET https://my.matterport.com/api/player/models/{model_id}
```
- Retorna informações básicas sobre o modelo

```
GET https://my.matterport.com/api/v1/models/{model_id}/sweeps
```
- Retorna dados sobre sweeps (panoramas) disponíveis

```
GET https://my.matterport.com/api/v1/models/{model_id}/mesh
```
- Retorna informações sobre a malha 3D

### 4.2 APIs GraphQL

O Matterport também usa uma API GraphQL para consultas mais complexas:

```
POST https://my.matterport.com/api/graph
```

Exemplo de consulta para obter informações detalhadas do modelo:
```graphql
query GetShowcaseData($id: ID!) {
  model(id: $id) {
    id
    name
    sweeps {
      id
      position
      rotation
      neighbors
    }
    floors {
      id
      name
      sweeps
    }
    mattertags {
      id
      position
      normal
      title
      description
      media {
        type
        src
      }
    }
  }
}
```

### 4.3 CDN de Recursos

Os recursos estáticos como panoramas, texturas e modelos 3D são servidos a partir de CDNs:

```
https://cdn-1.matterport.com/{model_id}/sweep/{sweep_id}/high_res.jpg
https://cdn-2.matterport.com/{model_id}/mesh/model.obj
```

## 5. Processos de Streaming e Otimização

O Matterport implementa diversas técnicas de streaming e otimização para garantir performance em diferentes dispositivos:

### 5.1 Streaming Progressivo

```javascript
// Pseudocódigo do sistema de streaming
class StreamingManager {
  constructor() {
    this.resourceQueue = [];
    this.loadingResources = new Set();
    this.maxConcurrentLoads = 4;
  }
  
  queueResource(resource, priority) {
    this.resourceQueue.push({ resource, priority });
    this.resourceQueue.sort((a, b) => b.priority - a.priority);
    this.processQueue();
  }
  
  async processQueue() {
    if (this.loadingResources.size >= this.maxConcurrentLoads) return;
    
    const nextResource = this.resourceQueue.shift();
    if (!nextResource) return;
    
    this.loadingResources.add(nextResource.resource);
    
    try {
      await this.loadResource(nextResource.resource);
    } finally {
      this.loadingResources.delete(nextResource.resource);
      this.processQueue();
    }
  }
  
  async loadResource(resource) {
    // Lógica para carregar recursos de diferentes tipos
    switch(resource.type) {
      case 'panorama':
        return this.loadPanorama(resource.id, resource.resolution);
      case 'mesh':
        return this.loadMeshChunk(resource.id);
      // outros tipos...
    }
  }
}
```

### 5.2 Carregamento Adaptativo

O sistema adapta a qualidade dos recursos com base na capacidade do dispositivo e na conexão:

```javascript
// Pseudocódigo do sistema adaptativo
function determineOptimalQuality() {
  const gpuTier = detectGPUCapabilities();
  const connectionSpeed = estimateConnectionSpeed();
  const deviceMemory = navigator.deviceMemory || 4;
  
  // Combinar fatores para determinar qualidade
  if (gpuTier === 'high' && connectionSpeed === 'fast' && deviceMemory >= 8) {
    return 'high';
  } else if (gpuTier === 'low' || connectionSpeed === 'slow' || deviceMemory <= 2) {
    return 'low';
  } else {
    return 'medium';
  }
}

function loadPanoramaAdaptive(sweepId) {
  const quality = determineOptimalQuality();
  
  switch(quality) {
    case 'high':
      return loadPanorama(sweepId, 'high_res.jpg');
    case 'medium':
      return loadPanorama(sweepId, 'medium_res.jpg');
    case 'low':
      return loadPanorama(sweepId, 'low_res.jpg');
  }
}
```

## 6. Detalhes do Sistema de Mapas de Profundidade

Os mapas de profundidade são cruciais para várias funcionalidades do Matterport:

### 6.1 Formato de Dados

Os mapas de profundidade são armazenados como imagens PNG de 16 bits:

- Cada pixel contém um valor de 0 a 65535
- A profundidade real é calculada usando uma função de conversão
- Tipicamente armazenados em resolução inferior aos panoramas RGB

### 6.2 Uso em Navegação e Interação

```javascript
// Pseudocódigo de uso de mapa de profundidade
function computeRealWorldPosition(screenX, screenY) {
  // Converter coordenadas de tela para coordenadas de textura panorâmica
  const textureU = screenX / window.innerWidth;
  const textureV = screenY / window.innerHeight;
  
  // Obter profundidade do mapa de profundidade
  const depthValue = getDepthValueFromTexture(currentDepthMap, textureU, textureV);
  
  // Converter para metros reais
  const realDepth = convertDepthValueToMeters(depthValue);
  
  // Calcular direção do raio a partir da câmera
  const ray = calculateRayFromCamera(camera, screenX, screenY);
  
  // Calcular posição 3D multiplicando direção pela profundidade
  const position = new THREE.Vector3().copy(camera.position).add(
    ray.multiplyScalar(realDepth)
  );
  
  return position;
}

function handleMeasurementToolClick(event) {
  const position = computeRealWorldPosition(event.clientX, event.clientY);
  
  if (measurementPoints.length === 0) {
    // Primeiro ponto da medição
    addMeasurementPoint(position);
  } else {
    // Segundo ponto, calcular distância
    const previousPoint = measurementPoints[measurementPoints.length - 1];
    const distance = position.distanceTo(previousPoint);
    
    // Mostrar distância em metros
    displayMeasurement(distance);
    
    // Adicionar linha visual entre pontos
    addMeasurementLine(previousPoint, position);
    
    // Adicionar segundo ponto
    addMeasurementPoint(position);
  }
}
```

## 7. Mecanismos de Interação e Ferramentas

### 7.1 Sistema de Medição

O sistema de medição utiliza os mapas de profundidade e raycasting para permitir medições precisas:

```javascript
// Pseudocódigo da ferramenta de medição
class MeasurementTool {
  constructor(scene, camera, depthManager) {
    this.scene = scene;
    this.camera = camera;
    this.depthManager = depthManager;
    this.points = [];
    this.lines = [];
    this.active = false;
    
    this.lineGeometry = new THREE.BufferGeometry();
    this.lineMaterial = new THREE.LineBasicMaterial({ color: 0xff0000 });
    this.measurementLine = new THREE.Line(this.lineGeometry, this.lineMaterial);
    this.scene.add(this.measurementLine);
  }
  
  activate() {
    this.active = true;
    this.points = [];
    this.updateLine();
  }
  
  addPoint(screenPosition) {
    const worldPosition = this.depthManager.screenToWorld(
      screenPosition.x, 
      screenPosition.y
    );
    
    if (worldPosition) {
      this.points.push(worldPosition);
      this.updateLine();
      
      if (this.points.length === 2) {
        return this.calculateDistance();
      }
    }
    
    return null;
  }
  
  calculateDistance() {
    if (this.points.length !== 2) return null;
    
    const distance = this.points[0].distanceTo(this.points[1]);
    return distance; // em metros
  }
  
  updateLine() {
    if (this.points.length < 2) {
      this.lineGeometry.setFromPoints([]);
      return;
    }
    
    this.lineGeometry.setFromPoints(this.points);
    this.lineGeometry.attributes.position.needsUpdate = true;
  }
}
```

### 7.2 Sistema de Mattertags

Mattertags são anotações ancoradas em pontos 3D específicos:

```javascript
// Pseudocódigo do sistema de Mattertags
class MattertagSystem {
  constructor(scene, camera, modelData) {
    this.scene = scene;
    this.camera = camera;
    this.tags = [];
    
    // Carregar tags a partir dos dados do modelo
    modelData.mattertags.forEach(tagData => {
      this.createTag(tagData);
    });
  }
  
  createTag(tagData) {
    // Criar elemento visual 3D
    const sprite = new THREE.Sprite(
      new THREE.SpriteMaterial({
        map: tagIconTexture,
        color: 0xffffff
      })
    );
    
    sprite.position.set(
      tagData.position[0],
      tagData.position[1],
      tagData.position[2]
    );
    
    // Escalar adequadamente
    sprite.scale.set(0.5, 0.5, 0.5);
    
    // Adicionar à cena
    this.scene.add(sprite);
    
    // Armazenar informações
    const tag = {
      id: tagData.id,
      sprite: sprite,
      data: tagData,
      selected: false
    };
    
    this.tags.push(tag);
    return tag;
  }
  
  update() {
    // Atualizar visibilidade e tamanho de tags com base na distância
    this.tags.forEach(tag => {
      // Calcular distância da câmera
      const distance = this.camera.position.distanceTo(tag.sprite.position);
      
      // Ajustar escala com base na distância
      const scale = Math.min(0.5, 0.1 + 0.4 / Math.max(1, distance / 5));
      tag.sprite.scale.set(scale, scale, scale);
      
      // Verificar visibilidade
      tag.sprite.visible = distance < 30; // só mostrar tags próximas
    });
  }
  
  handleClick(raycaster) {
    // Verificar interseção com tags
    const intersects = raycaster.intersectObjects(
      this.tags.map(tag => tag.sprite)
    );
    
    if (intersects.length > 0) {
      const clickedTag = this.getTagBySprite(intersects[0].object);
      this.selectTag(clickedTag);
      return true;
    }
    
    return false;
  }
  
  selectTag(tag) {
    // Mostrar interface com informações da tag
    showTagInfo(tag.data);
    
    // Destacar tag selecionada
    this.tags.forEach(t => {
      t.selected = (t.id === tag.id);
      t.sprite.material.color.set(t.selected ? 0xffff00 : 0xffffff);
    });
  }
}
```

## 8. Conclusões da Engenharia Reversa

Através da análise do matterport-dl e dos repositórios do GitHub, podemos concluir:

1. **Arquitetura**: O Matterport é construído como uma aplicação web WebGL com forte ênfase em streaming e otimização de recursos 3D.

2. **Dados**: O sistema organiza dados em vários níveis de detalhe (panoramas, nuvens de pontos, meshes) para permitir carregamento progressivo.

3. **Renderização**: Combina renderização de panoramas (para navegação imersiva) com renderização de malhas 3D (para visões dollhouse e floorplan).

4. **Performance**: Implementa múltiplas estratégias de otimização, como carregamento adaptativo, streaming progressivo e caching.

5. **Extensibilidade**: O sistema tem uma arquitetura modular que permite extensões como Mattertags, medições e outros plug-ins.

6. **Tecnologias**: Baseia-se principalmente em WebGL/Three.js para renderização, com JavaScript/TypeScript para lógica de aplicação.

Esta análise de engenharia reversa fornece insights valiosos sobre como o Matterport implementa sua solução de digitalização e visualização 3D, e pode servir de base para o desenvolvimento de sistemas similares ou complementares.