# JavaScript que se Ejecuta en el Navegador - TechModa

## 📝 Archivo env-config.js (inyectado en runtime)

```javascript
// frontend/dist/env-config.js
// Generado automáticamente durante el deploy

window.__ENV = {
  VITE_API_URL: 'https://wkhzf6meg52zmd6xyo2gvxne6m0ucyxk.lambda-url.us-east-1.on.aws'
};

window.__LAMBDA_URLS = {
  translateUrl: 'https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/',
  voiceUrl: 'https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/'
};
```

---

## 🔗 Flujo de Peticiones HTTP

### 1️⃣ Al Cargar la Página - Obtener Productos

```javascript
// GET /products (desde api.ts)
const response = await fetch('https://wkhzf6meg52zmd6xyo2gvxne6m0ucyxk.lambda-url.us-east-1.on.aws/products');
const data = await response.json();
// Retorna: { products: [Product[], ...] }
```

**Respuesta esperada:**
```json
{
  "products": [
    {
      "productId": "78499f93-187e-493a-bff1-46963ce1e590",
      "name": "Tenis blancos minimalistas",
      "description": "Sneakers de cuero sintético blanco, suela de goma.",
      "price": 74.50,
      "category": "Zapatos",
      "stock": 25,
      "imageUrl": "https://techmoda-ai-li-frontend.s3.us-east-1.amazonaws.com/assets/tenis-blancos.jpg",
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    // ... más productos
  ]
}
```

---

### 2️⃣ Traducir Producto - POST a Lambda de Translate

```javascript
// ProductAIFeatures.tsx - handleTranslate()
const translateUrl = window.__LAMBDA_URLS.translateUrl; // 'https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/'
const productId = '78499f93-187e-493a-bff1-46963ce1e590';
const targetLang = 'en';

const response = await fetch(
  `${translateUrl.replace(/\/+$/, '')}?id=${productId}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ target: targetLang })
  }
);

const data = await response.json();
```

**URL que se envía:**
```
POST https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/?id=78499f93-187e-493a-bff1-46963ce1e590
Content-Type: application/json

{
  "target": "en"
}
```

**Respuesta esperada:**
```json
{
  "productId": "78499f93-187e-493a-bff1-46963ce1e590",
  "target": "en",
  "translation": {
    "name": "Minimalist White Sneakers",
    "description": "White synthetic leather sneakers, rubber sole."
  }
}
```

**Lo que sucede en el backend (Lambda Python):**
1. Recibe `?id=` en queryStringParameters
2. Obtiene el producto de DynamoDB
3. Llama a `boto3.client('translate').translate_text()`
   - Amazon Translate detecta idioma origen = 'es'
   - Traduce name y description a 'en'
4. Guarda translations.en en DynamoDB
5. Retorna el JSON

---

### 3️⃣ Generar Audio - POST a Lambda de Polly

```javascript
// ProductAIFeatures.tsx - handleGenerateAudio()
const voiceUrl = window.__LAMBDA_URLS.voiceUrl; // 'https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/'
const productId = '78499f93-187e-493a-bff1-46963ce1e590';
const language = 'es'; // o 'en'

const response = await fetch(
  `${voiceUrl.replace(/\/+$/, '')}?id=${productId}`,
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ lang: language })
  }
);

const data = await response.json();
if (data.audioUrl) {
  audioRef.current.src = data.audioUrl;
  audioRef.current.play();
}
```

**URL que se envía:**
```
POST https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/?id=78499f93-187e-493a-bff1-46963ce1e590
Content-Type: application/json

{
  "lang": "es"
}
```

**Respuesta esperada:**
```json
{
  "productId": "78499f93-187e-493a-bff1-46963ce1e590",
  "lang": "es",
  "voice": "Lupe",
  "audioUrl": "https://techmoda-ai-li-audio.s3.us-east-1.amazonaws.com/audio/78499f93-187e-493a-bff1-46963ce1e590-es.mp3?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...",
  "expiresIn": 3600
}
```

**Lo que sucede en el backend (Lambda Python):**
1. Recibe `?id=` en queryStringParameters y `lang` en body
2. Obtiene el producto de DynamoDB
3. Si lang='en' y hay traducción, usa traducción; sino usa original
4. Llama a `boto3.client('polly').synthesize_speech()`
   - Genera audio MP3 con voces: 'Lupe' (ES) o 'Joanna' (EN)
5. Sube el MP3 a S3 (bucket privado)
6. Genera URL prefirmada con validez de 3600 segundos
7. Guarda aiAudioUrl en DynamoDB
8. Retorna el JSON

---

## 🔐 Cabeceras HTTP (CORS)

**Enviadas por el navegador (automatic):**
```
Origin: https://d2eiybqber8s8k.cloudfront.net
```

**Retornadas por Lambda (en el código Python):**
```javascript
// del archivo app.py de cada sesión
headers: {
  "Content-Type": "application/json",
  "Access-Control-Allow-Origin": "*"
}
```

⚠️ **Importante:** No incluir `Cors:` en `FunctionUrlConfig` del template.yaml (causaba cabeceras duplicadas).

---

## 📊 Estado Local en ProductAIFeatures

```typescript
const [language, setLanguage] = useState<'es' | 'en'>('es');
// Controla qué idioma se muestra

const [loadingTranslate, setLoadingTranslate] = useState(false);
// true mientras está haciendo fetch a Lambda de Translate

const [loadingAudio, setLoadingAudio] = useState(false);
// true mientras está haciendo fetch a Lambda de Polly

const [error, setError] = useState<string | null>(null);
// Almacena el error si falla alguna petición

const audioRef = useRef<HTMLAudioElement>(null);
// Referencia al elemento <audio> para controlar reproducción
```

---

## 🎯 Selección de Idioma

```javascript
// En ProductAIFeatures
const currentName = language === 'en'
  ? product.aiTranslations?.en?.name || product.name
  : product.name;

const currentDesc = language === 'en'
  ? product.aiTranslations?.en?.description || product.description
  : product.description;
```

**Lógica:**
- Si seleccionó **EN** y `product.aiTranslations.en` existe → muestra traducción
- Si seleccionó **EN** pero no hay traducción → muestra original (en español)
- Si seleccionó **ES** → siempre muestra original

---

## 🎬 Evento: Clic en "Traducir"

```typescript
const handleTranslate = async () => {
  const targetLang = language === 'es' ? 'en' : 'es';
  setLoadingTranslate(true);
  setError(null);

  try {
    if (!translateUrl) {
      throw new Error('Translate URL no configurada');
    }

    // 1. Hacer la petición
    const response = await fetch(
      `${translateUrl.replace(/\/+$/, '')}?id=${product.productId}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ target: targetLang }),
      }
    );

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`Error ${response.status}: ${errorText}`);
    }

    // 2. Parsear respuesta
    const data = await response.json();

    // 3. Cambiar idioma
    setLanguage(targetLang);

    // 4. Actualizar producto en el componente padre
    if (onProductUpdated) {
      const updated = {
        ...product,
        aiTranslations: {
          ...product.aiTranslations,
          [targetLang]: data.translation,
        },
      };
      onProductUpdated(updated);
    }
  } catch (err) {
    setError(err instanceof Error ? err.message : 'Error al traducir');
  } finally {
    setLoadingTranslate(false);
  }
};
```

---

## 🔊 Evento: Clic en "Escuchar descripción"

```typescript
const handleGenerateAudio = async () => {
  setLoadingAudio(true);
  setError(null);

  try {
    if (!voiceUrl) {
      throw new Error('Voice URL no configurada');
    }

    // 1. Hacer la petición
    const response = await fetch(
      `${voiceUrl.replace(/\/+$/, '')}?id=${product.productId}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ lang: language }),
      }
    );

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`Error ${response.status}: ${errorText}`);
    }

    // 2. Parsear respuesta
    const data = await response.json();

    // 3. Asignar URL de audio al reproductor
    if (audioRef.current && data.audioUrl) {
      audioRef.current.src = data.audioUrl;
      audioRef.current.play(); // Opcionalmente, iniciar reproducción
    }
  } catch (err) {
    setError(err instanceof Error ? err.message : 'Error al generar audio');
  } finally {
    setLoadingAudio(false);
  }
};
```

---

## 🎵 El Reproductor de Audio

```html
<audio
  ref={audioRef}
  controls
  className="w-full h-8"
  style={{ marginTop: '0.5rem' }}
/>
```

**Atributos HTML nativos:**
- `controls` → muestra play, pause, progress bar, volumen
- `ref={audioRef}` → acceso desde React para .play(), .pause(), asignar .src

**Interacciones del usuario:**
```javascript
// Reproducir
audioRef.current.play();

// Pausar
audioRef.current.pause();

// Cambiar volumen
audioRef.current.volume = 0.5; // 0 a 1

// Obtener duración
const duration = audioRef.current.duration; // en segundos
```

---

## ❌ Manejo de Errores Comunes

### CORS Error
```
Access to fetch at '...' from origin '...' has been blocked by CORS policy:
The 'Access-Control-Allow-Origin' header contains multiple values
```

**Causa:** Cabeceras CORS duplicadas (template.yaml + código)
**Solución:** Remover `Cors:` del `FunctionUrlConfig` en template.yaml

---

### 404 Not Found
```
POST https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/?id=...
Status: 404
```

**Causa:** URL de Lambda mal formada o producto no existe
**Solución:** Verificar `?id=` está incluido; verificar `product.productId` es válido

---

### Translate URL no configurada
```
Error: Translate URL no configurada
```

**Causa:** `window.__LAMBDA_URLS` no cargó correctamente
**Solución:** Verificar `frontend/dist/env-config.js` tiene las URLs inyectadas

---

## 📡 Logs de Consola (DevTools F12)

```javascript
// En ProductAIFeatures.tsx
console.log('Translate URL:', translateUrl);
console.log('Voice URL:', voiceUrl);
console.log('Product ID:', product.productId);

// Cuando hace fetch
console.log('Fetching translate...', url);

// Respuesta
console.log('Translation response:', data);
console.log('Audio URL:', data.audioUrl);
```

**Para ver logs:**
1. Abre DevTools: F12
2. Pestaña **Console**
3. Ejecuta acciones en la página
4. Mira los logs

---

## 🚀 Flujo Completo Usuario

```
1. Página carga
   ↓
2. ProductCard renderiza para cada producto
   ↓
3. ProductAIFeatures monta
   → Lee window.__LAMBDA_URLS
   → Muestra botones ES/EN/Traducir/Escuchar
   ↓
4. Usuario hizo clic EN
   → setLanguage('en')
   → Aparece botón "Traducir" activo
   ↓
5. Usuario hizo clic Traducir
   → handleTranslate() inicia
   → Envía POST a Lambda Translate
   → Recibe traducción
   → setLanguage('en')
   → onProductUpdated({ ...aiTranslations.en })
   → Muestra texto en inglés
   ↓
6. Usuario hizo clic Escuchar descripción
   → handleGenerateAudio() inicia
   → Envía POST a Lambda Polly
   → Recibe audioUrl prefirmada (3600s)
   → audioRef.current.src = audioUrl
   → Reproductor se llena con audio
   ↓
7. Usuario presiona ▶
   → audioRef.current.play()
   → Escucha la descripción en voz
```

---

## 📝 ResumenConfiguraciones Necesarias

```javascript
// frontend/dist/env-config.js (obligatorio)
window.__ENV = {
  VITE_API_URL: 'https://...'
};
window.__LAMBDA_URLS = {
  translateUrl: 'https://...',
  voiceUrl: 'https://...'
};

// Inyectadas por: scripts/inject-env.sh
// Durante: bash scripts/deploy-frontend.sh
```
