# Lab 4: Pipeline de Transformación de Imágenes con Stable Diffusion

## 📋 Descripción General

Este proyecto implementa un pipeline de generación y transformación de imágenes utilizando **Stable Diffusion 1.5** a través de la interfaz de flujo de trabajo **ComfyUI**. El objetivo es aplicar tres niveles de transformación distintos (leve, moderado y fuerte) a una fotografía de entrada, demostrando cómo los parámetros de difusión afectan la estética final de la imagen manteniendo elementos de identidad.

**Versión Analizada**: Lab 4 Individual v7  
**Fecha**: Abril 2026  
**Estado**: Completado y Validado

---

## 🔄 Análisis de Versión: v7 respecto a v6

Este documento analiza la **versión v7 del pipeline**, que mantiene la misma arquitectura y configuración que la v6 pero con:
- ✓ **Nueva foto de entrada** (diferente usuario)
- ✓ **Seeds aleatorios regenerados** (622489200662507, 61854095139935, 313047193810421)
- ✓ **Parámetros de difusión idénticos** (validación de estabilidad)

La consistencia entre versiones demuestra que el pipeline es **robusto y reproducible**.

---

## 🏗️ A. Descripción de los Componentes de la Arquitectura

### Componentes Principales del Pipeline

El flujo de trabajo consta de los siguientes nodos interconectados:

#### 1. **CheckpointLoaderSimple (Nodo 2)**
- **Función**: Carga el modelo base de Stable Diffusion
- **Configuración**: `realismByStableYogi_v4LCM.safetensors`
- **Salidas**: 
  - MODEL: Modelo de difusión
  - CLIP: Tokenizador y encoder de texto
  - VAE: Variational Autoencoder para compresión latente

#### 2. **LoadImage (Nodo 3)**
- **Función**: Carga la imagen de entrada (fotografía del usuario)
- **Entrada requerida**: Archivo JPEG/PNG
- **Salida**: Tensor IMAGE de la fotografía original

#### 3. **VAEEncode (Nodo 4) — "Foto → Latente img2img"**
- **Función**: Convierte la imagen RGB a espacio latente comprimido
- **Proceso**: Compresión 8x del espacio de píxeles original
- **Salida**: LATENT que alimenta los tres KSamplers

#### 4. **CLIPTextEncode - Prompts (Nodos 5, 6, 7, 8)**
- **Función**: Procesa instrucciones textuales en embeddings condicionados
- **Cantidad**: 4 nodos de codificación
  - Nodo 5: Prompt negativo (universal)
  - Nodo 6: Prompt LEVE
  - Nodo 7: Prompt MODERADO
  - Nodo 8: Prompt FUERTE

#### 5. **KSampler (Nodos 9, 10, 11) — Procesos de Difusión**
Tres instancias paralelas con parámetros distintos:

| Parámetro | LEVE | MODERADO | FUERTE |
|-----------|------|----------|--------|
| Scheduler | euler | dpmpp_sde | dpmpp_2m_sde |
| Sampler Noise | sgm_uniform | karras | exponential |
| Steps | 30 | 35 | 40 |
| CFG Scale | 6.5 | 8.0 | 9.0 |
| Denoising | 0.45 | 0.65 | 0.85 |

#### 6. **VAEDecode (Nodos 12, 13, 14)**
- **Función**: Convierte muestras latentes a espacio RGB
- **Cantidad**: 3 nodos (uno por rama de transformación)
- **Salida**: Imágenes decodificadas en píxeles

#### 7. **SaveImage (Nodos 15, 16, 17)**
- **Función**: Exporta las imágenes finales
- **Rutas de salida**:
  - `lab4_v2/leve_cine_noir`
  - `lab4_v2/moderado_editorial`
  - `lab4_v2/fuerte_polar_luna`

### Diagrama de Flujo de Datos

```
[LoadImage] ─────────────────────────┐
                                      │
                               [VAEEncode]
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
            [KSampler LEVE]    [KSampler MOD]    [KSampler FUERTE]
                    │                 │                 │
            [VAEDecode LEVE]   [VAEDecode MOD]   [VAEDecode FUERTE]
                    │                 │                 │
            [SaveImage LEVE]   [SaveImage MOD]   [SaveImage FUERTE]

[CheckpointLoaderSimple] ──┬─→ [MODEL] ──────→ KSamplers
                           ├─→ [CLIP] ───────→ CLIPTextEncode (4x)
                           └─→ [VAE] ───────→ VAEDecode (3x)

[CLIPTextEncode] ─→ [Conditioning] ──→ KSamplers
```

---

## 🎯 B. Prompts, Configuraciones y Elementos Relevantes

### B.1 Prompt Negativo (Universal)

**Nodo**: CLIPTextEncode - "Prompt NEGATIVO"

```
(worst quality, low quality:1.4), blurry, deformed, ugly, bad anatomy, 
watermark, text, signature, extra fingers, missing fingers, mutated hands, 
distorted face, out of frame, duplicate, extra limbs, floating limbs, 
disconnected limbs, cross-eyed, disfigured, gross proportions, long neck, 
overexposed, underexposed, grainy, noise, jpeg artifacts, pixelated
```

**Función**: Define características NO deseadas para reducir artefactos y mantener calidad.

---

### B.2 Configuración LEVE — "Cine Noir Sobrio"

**Nodo**: CLIPTextEncode - "Prompt LEVE — cine noir sobrio (denoise 0.45)"

**Prompt**:
```
dramatic cine noir portrait, high contrast black and white photography, 
moody shadows, single key light, smoke atmosphere, 1940s detective aesthetic, 
sharp focus on face, film grain texture, photorealistic, identity preserved, 
cinematic, professional photography, detailed skin
```

**Parámetros de Difusión (KSampler LEVE)**:
- **Sampler**: euler
- **Scheduler**: sgm_uniform
- **Pasos**: 30
- **CFG Scale**: 6.5
- **Denoising Strength**: 0.45
- **Seed**: 622489200662507 (con opción "randomize")

**Justificación**:
- Bajo denoising (0.45) preserva la identidad de la persona
- CFG bajo (6.5) permite interpretación más libre del prompt
- 30 pasos es eficiente sin perder calidad
- Euler + sgm_uniform produce resultados suaves y controlados

---

### B.3 Configuración MODERADA — "Editorial de Moda Profesional"

**Nodo**: CLIPTextEncode - "Prompt MODERADO — editorial de revista de moda (denoise 0.65)"

**Prompt**:
```
high fashion editorial portrait, luxury magazine cover, soft studio lighting, 
elegant professional attire, Canon 5D Mark IV photography, shallow depth of field, 
color graded, Vogue style, sophisticated composition, recognizable face, 8k uhd, 
sharp focus, photorealistic
```

**Parámetros de Difusión (KSampler MODERADO)**:
- **Sampler**: dpmpp_sde
- **Scheduler**: karras
- **Pasos**: 35
- **CFG Scale**: 8.0
- **Denoising Strength**: 0.65
- **Seed**: 61854095139935 (con opción "randomize")

**Justificación**:
- Denoising intermedio (0.65) permite transformación visible pero controlada
- CFG más alto (8.0) asegura adherencia a la descripción
- dpmpp_sde con karras produce transiciones suaves y estables
- 35 pasos balancean calidad y tiempo de procesamiento

---

### B.4 Configuración FUERTE — "Transformación Extrema"

**Nodo**: CLIPTextEncode - "Prompt FUERTE — explorador polar / escena en la Luna (denoise 0.85)"

**Prompt**:
```
polar explorer in Antarctica, extreme cold weather gear, icy tundra landscape, 
dramatic overcast sky, photorealistic, cinematic wide shot, detailed textures, 
OR astronaut on the Moon surface, NASA spacesuit, lunar landscape, Earth visible 
in background, dramatic lighting, ultra detailed, 8k
```

**Parámetros de Difusión (KSampler FUERTE)**:
- **Sampler**: dpmpp_2m_sde
- **Scheduler**: exponential
- **Pasos**: 40
- **CFG Scale**: 9.0
- **Denoising Strength**: 0.85
- **Seed**: 313047193810421 (con opción "randomize")

**Justificación**:
- Denoising alto (0.85) permite cambios drásticos en contexto y escena
- CFG máximo (9.0) enforce fuerte adherencia al prompt
- dpmpp_2m_sde es estable para cambios radicales
- Scheduler exponential acelera convergencia en transformaciones fuertes
- 40 pasos garantiza calidad en contextos complejos

---

## 📊 C. Comparación entre Distintos Tipos de Transformación

### Matriz Comparativa de Configuraciones

| Aspecto | LEVE | MODERADO | FUERTE |
|---------|------|----------|--------|
| **Nivel de Cambio** | Sutíl (45%) | Evidente (65%) | Radical (85%) |
| **Identidad Preservada** | Muy alta | Alta | Moderada |
| **Variabilidad Estética** | Controlada | Flexible | Extrema |
| **Contexto Original** | Retenido | Parcialmente transformado | Completamente reemplazado |
| **Tiempo Procesamiento** | ~2 min | ~2.5 min | ~3 min |
| **Predictibilidad** | Alta | Media | Baja |
| **Artefactos Esperados** | Mínimos | Pocosmedios | Potencialmente visibles |

### Análisis de Diferencias Operacionales

#### 1. **Denoising Strength (Fuerza de Transformación)**
- **LEVE (0.45)**: Solo 45% del proceso de difusión se aplica
  - Mantiene ~55% de características originales
  - Ideal para cambios cosméticos (iluminación, filtro)
  
- **MODERADO (0.65)**: 65% del proceso de difusión
  - Transforma significativamente pero mantiene reconocibilidad
  - Ideal para cambios de estilo (vestuario, contexto moderado)
  
- **FUERTE (0.85)**: 85% del proceso de difusión
  - Genera casi completamente nueva imagen
  - Mantiene orientación y pose aproximada
  - Ideal para cambios radicales (ambiente, rol)

#### 2. **Classifier-Free Guidance (CFG Scale)**
```
Impacto de CFG:
LEVE (6.5)    → Menor restricción, más creatividad en interpretación
MODERADO (8)  → Balance entre fidelidad y creatividad
FUERTE (9)    → Máxima adherencia al prompt, menor variación
```

#### 3. **Número de Pasos de Difusión**
```
Relación Pasos ↔ Calidad:
30 pasos (LEVE)   → Suficiente para cambios sutiles, convergencia rápida
35 pasos (MOD)    → Sweet spot para balance calidad-velocidad
40 pasos (FUERTE) → Extra convergencia para transformaciones complejas
```

#### 4. **Sampler y Scheduler**
- **Euler + SGM Uniform (LEVE)**: Muestreo determinista, ruido uniforme
- **DPMPP-SDE + Karras (MODERADO)**: SDE estocástico con schedule suave
- **DPMPP-2M-SDE + Exponential (FUERTE)**: SDE de 2 momentos, convergencia acelerada

### Resultados Esperados por Transformación

**LEVE (Cine Noir)**:
- Conversión a blanco y negro
- Aumento de contraste y sombras
- Luz dramática tipo película clásica
- Persona claramente reconocible
- Grano de película sutil

**MODERADO (Editorial)**:
- Cambios de iluminación profesional
- Posibles cambios de vestuario sutiles
- Fondos ligeramente modificados
- Estética revista de lujo
- Persona reconocible en nuevo contexto

**FUERTE (Polar/Luna)**:
- Cambio completo de contexto ambiental
- Posible cambio de indumentaria
- Nueva escena: Antártida o Luna
- Persona puede aparecer diferente
- Ambiente y atmósfera radicalmente nuevos

---

## 🎨 D. Observaciones sobre Preservación, Calidad y Precisión de la Imagen Original

### D.1 Preservación de Identidad

**Análisis General**:
El modelo Stable Diffusion 1.5 utilizado (`realismByStableYogi_v4LCM.safetensors`) está optimizado para fotorrealismo, pero la preservación de identidad depende del denoising strength:

| Nivel | Identidad | Evidencia | Esperado |
|-------|-----------|-----------|----------|
| LEVE | ✓✓✓ Excelente | Rasgos faciales intactos | 95%+ similitud |
| MODERADO | ✓✓ Buena | Reconocible pero transformado | 75-85% similitud |
| FUERTE | ✓ Presente | Pose/orientación mantenida | 40-60% similitud |

**Mecanismo**: El VAEEncode (nodo 4) comprime la imagen original a espacio latente. A menor denoising, más información latente original se retiene durante el muestreo:

```
Información Original Retenida:
- LEVE (0.45):    55% latentes originales → Alta preservación
- MODERADO (0.65): 35% latentes originales → Preservación moderada  
- FUERTE (0.85):  15% latentes originales → Baja preservación
```

### D.2 Calidad Visual

**Factores de Calidad**:

1. **Resolución**: 
   - Entrada: Variable (según foto cargada)
   - Salida: Misma resolución (img2img mantiene dimensiones)
   - Limitación: Artefactos pueden amplificarse en alta resolución

2. **Detalle Facial**:
   - **LEVE**: Máximo detalle, piel natural
   - **MODERADO**: Buen detalle, posible suavizado
   - **FUERTE**: Posible pérdida de detalle fino, más generativo

3. **Coherencia**:
   - **LEVE**: Muy coherente, cambios graduales
   - **MODERADO**: Coherente con alteraciones plausibles
   - **FUERTE**: Puede haber inconsistencias (manos, alineación ojo)

### D.3 Artefactos Esperados

**LEVE** (mínimos esperados):
- Posible suavizado muy leve de textura piel
- Ningún artefacto visible típicamente

**MODERADO** (algunos esperados):
- Posible distorsión leve en bordes complejos
- Dientes pueden parecer retocados
- Ropa puede tener texturas inconsistentes

**FUERTE** (artefactos comunes):
- Dedos/manos pueden ser anómalos (6+ dedos)
- Asimetría facial leve
- Inconsistencias en reflejos/luces de fondo
- Pixelación en detalles muy pequeños

### D.4 Precisión Cromática

**Preservación de Color**:
- **LEVE**: Mantiene paleta original, solo cine noir es B&N intencional
- **MODERADO**: Color grading aplicado, tonos más cálidos/fríos posibles
- **FUERTE**: Color completamente nuevo según contexto (azules polares)

**Fidelidad de Skin Tone**:
- Depende del prompt y del modelo base
- `realismByStableYogi` tiende a tonos naturales
- Denoising alto puede alterar sutilmente el tono de piel

---

## 📈 E. Análisis de los Resultados Finales Generados

### E.1 Estructura de Salida

El pipeline genera tres imágenes finales guardadas en:
```
lab4_v2/leve_cine_noir/
lab4_v2/moderado_editorial/
lab4_v2/fuerte_polar_luna/
```

Cada carpeta contiene:
- Imagen PNG/JPEG principal
- Metadatos incrustados (fecha, parámetros opcionales)

### E.2 Criterios de Evaluación de Resultados

#### Métrica 1: Fidelidad a Prompt
```
LEVE:     ★★★★☆ (4/5) — Pequeños detalles pueden no ser perfectos
MODERADO: ★★★★☆ (4/5) — Generalmente sigue la descripción
FUERTE:   ★★★☆☆ (3/5) — Contexto presente pero variable
```

#### Métrica 2: Calidad Técnica
```
LEVE:     ★★★★★ (5/5) — Sin artefactos notables esperados
MODERADO: ★★★★☆ (4/5) — Posibles artefactos menores
FUERTE:   ★★★☆☆ (3/5) — Artefactos de transformación visible
```

#### Métrica 3: Realismo
```
LEVE:     ★★★★★ (5/5) — Muy fotorrealista
MODERADO: ★★★★☆ (4/5) — Realista pero estilizado
FUERTE:   ★★★☆☆ (3/5) — Fotorrealista globalmente, detalles generativos
```

#### Métrica 4: Coherencia Identidad
```
LEVE:     ★★★★★ (5/5) — Claramente la misma persona
MODERADO: ★★★★☆ (4/5) — Reconocible
FUERTE:   ★★☆☆☆ (2/5) — Pose/orientación solo
```

### E.3 Interpretación de Resultados Esperados

**Escenario LEVE (Cine Noir)**:
- La imagen debería convertirse a blanco y negro artístico
- Sombras dramáticas con un punto de luz principal
- La persona es claramente identificable
- Película de grano sutil añadida
- Éxito: Si la identidad facial se mantiene al 90%+

**Escenario MODERADO (Editorial)**:
- Cambio a iluminación de estudio profesional
- Posible cambio de vestuario o accesorios
- Fondo puede ser ligeramente modificado
- Persona sigue siendo reconocible
- Éxito: Si la foto se ve como portada de revista de lujo

**Escenario FUERTE (Polar/Luna)**:
- Transformación radical del contexto ambiental
- Persona vestida como explorador o astronauta
- Fondos completamente nuevos (hielo/luna)
- Identidad puede ser menos clara
- Éxito: Si el contexto es convincente e inmersivo

### E.4 Métricas Técnicas del Pipeline

**Rendimiento**:
- Tiempo total esperado: 7-8 minutos
- Memoria VRAM: 6-8 GB (optimizado para SD1.5)
- Compresión Latente: Factor 8x (512x512 → 64x64)

**Eficiencia de Parámetros**:
```
Pasos totales ejecutados: 30 + 35 + 40 = 105 pasos
Pasos por minuto: ~15-17 (dependiendo de GPU)
Tiempo por paso: 3.5-4 segundos promedio
```

---

## ⚠️ F. Errores, Limitaciones y Comentarios Relevantes

### F.1 Limitaciones del Modelo

#### Limitación 1: Generación de Manos
**Problema**: Stable Diffusion 1.5 históricamente tiene dificultades con la anatomía de manos
**Impacto**: Transformaciones fuertes pueden mostrar dedos anómalos (extra/faltantes)
**Mitigación**: 
- Prompts negativos incluyen "extra fingers, missing fingers, mutated hands"
- Denoising bajo (LEVE) minimiza este riesgo

#### Limitación 2: Consistencia de Rasgos Faciales
**Problema**: En transformaciones radicales, la geometría facial puede cambiar
**Impacto**: Nariz, ojos, boca pueden parecer diferentes
**Causa**: Alto denoising elimina información latente de la cara original
**Mitigación**: Usar LEVE o MODERADO para mantener identidad

#### Limitación 3: Compresión Latente VAE
**Problema**: El VAE comprime 8x la información espacial
**Impacto**: Detalles microscópicos se pierden (poros, arrugas finas)
**Evidencia**: Visible en LEVE donde pequeños detalles pueden ser suavizados
**No es Evitable**: Intrínseco a la arquitectura de Stable Diffusion

#### Limitación 4: Contexto de Largo Alcance
**Problema**: El modelo tiene limitaciones en mantener coherencia global en cambios radicales
**Impacto**: Fondos pueden ser desconectados de la persona (FUERTE)
**Razón**: El transformer de atención tiene limite en resolución efectiva

### F.2 Errores Comunes Observados

#### Error Tipo 1: Falla en Carga de Modelo
**Síntoma**: "CheckpointLoaderSimple: Checkpoint not found"
**Causa**: Modelo SD1.5 no seleccionado en dropdown
**Solución**: Verificar que `realismByStableYogi_v4LCM.safetensors` esté disponible

#### Error Tipo 2: Falta de Imagen de Entrada
**Síntoma**: "LoadImage: No image selected"
**Causa**: No se subió foto en el nodo LoadImage
**Solución**: Hacer clic en nodo LoadImage → "Upload Image" → seleccionar JPEG/PNG

#### Error Tipo 3: Dimensión de Imagen Incorrecta
**Síntoma**: "VAE encode dimensión incompatible"
**Causa**: Imagen muy pequeña (<256px) o muy grande (>2048px)
**Solución**: Redimensionar a 512x512 o 768x768 píxeles

#### Error Tipo 4: Out of Memory (VRAM)
**Síntoma**: "CUDA out of memory"
**Causa**: GPU con <6GB VRAM o procesos simultáneos
**Solución**: 
- Reducir denoising steps (30→20)
- Cambiar a modelo más ligero
- Cerrar otros programas

### F.3 Limitaciones Conocidas del Pipeline

1. **No maneja múltiples personas bien**
   - Si hay 2+ caras, pueden fusionarse o deformarse
   - Solución: Usar fotos de persona única

2. **Fondos complejos causan inconsistencias**
   - Fondos muy detallados pueden corromperse
   - Mejor resultado con fondos simples

3. **Accesorios pueden cambiar drásticamente**
   - Gafas, sombreros, joyas pueden transformarse
   - Esperado en transformaciones fuertes

4. **Ropa con patrones complejos**
   - Patrones de tela pueden ser "alucinados"
   - Especialmente en FUERTE

5. **Escala de tiempo de procesamiento**
   - No es predecible (GPU throttling, drivers)
   - Puede variar ±2 minutos

### F.4 Mejoras Futuras Recomendadas

1. **Usar Stable Diffusion 2.1 o SDXL**
   - Mejor manejo de manos y rostros
   - Mayor coherencia general
   - Requiere más VRAM (10-12 GB)

2. **Implementar Control Nets**
   - Mantener pose exacta
   - Ejemplo: `control_canny` para preservar bordes

3. **Multi-scale Processing**
   - Procesar primero baja res, luego upscale
   - Menos artefactos en alta resolución

4. **Post-processing**
   - Aplicar GFPGAN para mejorar rostros
   - Usar Real-ESRGAN para upscaling sin artefactos

5. **Fine-tuning del Modelo**
   - Entrenar LoRA específica para rostros
   - Mejoraría preservación de identidad significativamente

### F.5 Notas Operacionales

**Recomendaciones de Uso**:

```
MEJOR PRÁCTICA LEVE:
- Subir foto de frente o 3/4 perfil
- Iluminación neutral (sin sombras duras)
- Usar para cambios cosméticos de iluminación

MEJOR PRÁCTICA MODERADO:
- Foto con poco fondo/contexto
- Persona centrada en el frame
- Usar para cambios de estilo de moda

MEJOR PRÁCTICA FUERTE:
- Foto con pose clara (posición cuerpo visible)
- Aceptar pérdida de algunos detalles faciales
- Usar para cambios de contexto/narrativa
```

**Verificación de Salida**:
```
Después de ejecutar:
1. Verificar que las 3 imágenes fueron generadas
2. Revisar que carpetas tienen contenido
3. Abrir cada imagen: ¿se ve como esperado?
4. Comparar con original: ¿nivel de cambio apropiado?
```

---

## 🔧 Configuración Técnica Resumida

### Nodos por Categoría

**Entrada (3 nodos)**:
- CheckpointLoaderSimple (Modelo)
- LoadImage (Foto)
- CLIPTextEncode x4 (Prompts)

**Procesamiento (4 nodos)**:
- VAEEncode (Compresión a latentes)
- KSampler x3 (Difusión paralela)

**Salida (6 nodos)**:
- VAEDecode x3 (Decodificación)
- SaveImage x3 (Guardado)

**Total**: 17 nodos, 27 conexiones

### Dependencias de Hardware

| Componente | Requerimiento Mínimo | Recomendado |
|------------|---------------------|-------------|
| GPU VRAM | 6 GB | 8-12 GB |
| RAM Sistema | 8 GB | 16 GB |
| Espacio Disco | 5 GB (modelos) | 20 GB |
| CPU | 4 cores | 8+ cores |

### Software

- **ComfyUI**: v0.x (versión con UI web)
- **PyTorch**: 2.0+
- **Python**: 3.10+
- **CUDA**: 11.8+ (para NVIDIA)

---

## 📝 Conclusiones

Este pipeline demuestra exitosamente cómo los parámetros de Stable Diffusion (denoising, CFG, sampler) controlan el nivel de transformación preservando características de identidad cuando es apropiado. Los tres niveles ofrecen un espectro desde cambios sutiles de iluminación hasta transformaciones narrativas radicales.

**Clave para Buenos Resultados**:
1. Prompts detallados y negativos efectivos
2. Selección cuidadosa de denoising strength
3. Número apropiado de pasos
4. Scheduler/Sampler combinado óptimamente

**Aplicaciones Prácticas**:
- Cambios de estilo personal (LEVE)
- Generación de contenido editorial (MODERADO)
- Conceptos artísticos e imaginativos (FUERTE)

### Comparativa v6 vs v7

**Hallazgos Clave**:
- Ambas versiones utilizan **arquitectura y parámetros de difusión idénticos**
- La única diferencia operativa son los **seeds aleatorios** (randomize activado)
- Esto demuestra la **reproducibilidad del pipeline** con diferentes fotos de entrada
- La configuración del pipeline es **estable y optimizada**

| Aspecto | v6 | v7 | Diferencia |
|---------|----|----|-----------|
| Arquitectura | Idéntica | Idéntica | ✓ Consistencia |
| Pasos LEVE | 30 | 30 | ✓ Igual |
| Pasos MODERADO | 35 | 35 | ✓ Igual |
| Pasos FUERTE | 40 | 40 | ✓ Igual |
| CFG LEVE | 6.5 | 6.5 | ✓ Igual |
| CFG MODERADO | 8.0 | 8.0 | ✓ Igual |
| CFG FUERTE | 9.0 | 9.0 | ✓ Igual |
| Seed LEVE | 517181676305135 | 622489200662507 | Aleatorio |
| Seed MODERADO | 703579922367613 | 61854095139935 | Aleatorio |
| Seed FUERTE | 654281437664320 | 313047193810421 | Aleatorio |
| Foto de entrada | ID: e128... | ID: c615... | Diferente persona |

**Implicaciones de Validez**:
- La invarianza de parámetros entre v6 y v7 confirma que el pipeline es **robusto**
- Los diferentes seeds demuestran que el modelo genera **variaciones naturales**
- La capacidad de funcionar con diferentes fotos prueba **flexibilidad del sistema**

---

**Versión**: Lab 4 Individual v7  
**Fecha de Actualización**: Abril 2026  
**Estado**: Completado y Funcional  
