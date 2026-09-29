# Estructura de Componentes React - TechModa

## 📋 Archivo HTML Base
**`frontend/index.html`**
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>TechModa Product Catalog API</title>
  </head>
  <body>
    <div id="root"></div>
    <script src="/env-config.js"></script>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## 🏗️ Árbol de Componentes

```
App.tsx (componente raíz)
├── Header
│   ├── Logo TechModa
│   └── Botón Admin Toggle
├── Search & Filter
│   ├── Input de búsqueda
│   └── Selector de categorías
├── ProductCard (repetido x3)
│   ├── Imagen del producto
│   ├── Nombre + Categoría
│   ├── Descripción
│   ├── Precio + Stock
│   ├── ProductAIFeatures
│   │   ├── Selector de idioma (ES/EN)
│   │   ├── Botón Traducir
│   │   ├── Vista previa de traducción
│   │   ├── Botón Escuchar descripción
│   │   └── Reproductor de audio
│   └── Botón Agregar al Carrito
└── Footer

ProductModal (modal de editar/crear)
├── Form inputs
│   ├── Nombre
│   ├── Descripción
│   ├── Precio
│   ├── Stock
│   ├── Categoría
│   └── URL de imagen
└── Botones Cancelar/Guardar
```

---

## 📁 Archivos de Componentes

### `src/App.tsx`
- **Props:** Ninguno
- **State:** 
  - `isAdmin`: boolean
  - `isModalOpen`: boolean
  - `searchTerm`: string
  - `categoryFilter`: string
  - `products`: Product[]
- **Funciones:**
  - `handleSaveProduct()` - Crear/actualizar producto
  - `handleEdit()` - Abrir modal de edición
  - `handleDelete()` - Eliminar producto
  - `handleCloseModal()` - Cerrar modal

---

### `src/components/ProductCard.tsx`
**Props:**
```typescript
interface ProductCardProps {
  product: Product;
  onEdit?: (product: Product) => void;
  onDelete?: (productId: string) => void;
  isAdmin?: boolean;
}
```

**State:**
- `currentProduct`: Product (para cambios locales)

**Estructura:**
```
<div> // Tarjeta
  <img> // Imagen del producto
  <div> // Contenido
    <h3> // Nombre + Categoría
    <p> // Descripción
    <div> // Precio + Stock
    <ProductAIFeatures /> // ← NUEVO
    <button> // Admin: Editar/Eliminar o Cliente: Agregar Carrito
  </div>
</div>
```

---

### `src/components/ProductAIFeatures.tsx` ⭐ NUEVO
**Props:**
```typescript
interface ProductAIFeaturesProps {
  product: Product;
  onProductUpdated?: (updated: Product) => void;
}
```

**State:**
- `language`: 'es' | 'en'
- `loadingTranslate`: boolean
- `loadingAudio`: boolean
- `error`: string | null
- `audioRef`: HTMLAudioElement

**Funciones:**
- `handleTranslate()` - Llama a Lambda de Translate
- `handleGenerateAudio()` - Llama a Lambda de Polly

**Estructura:**
```
<div> // Sección de AI Features
  {error && <div>Error: {error}</div>}
  
  <div> // Selector de idioma
    <button>ES</button>
    <button>EN</button>
    <button>Traducir</button>
  </div>
  
  <div> // Vista previa de traducción
    <p>{currentName}</p>
    <p>{currentDescription}</p>
  </div>
  
  <button> // Escuchar descripción
    🔊 Escuchar
  </button>
  
  <audio ref={audioRef} controls />
</div>
```

---

### `src/components/ProductModal.tsx`
**Props:**
```typescript
interface ProductModalProps {
  isOpen: boolean;
  onClose: () => void;
  onSave: (product: Omit<Product, 'productId' | 'createdAt' | 'updatedAt'>) => void;
  product?: Product;
}
```

**State:**
- `formData`: { name, description, price, category, stock, imageUrl }

---

### `src/hooks/useProducts.ts`
**Hook personalizado para gestionar productos:**
- `fetchProducts()` - GET /products
- `createProduct()` - POST /products
- `updateProduct()` - PUT /products/{id}
- `deleteProduct()` - DELETE /products/{id}

**Retorna:**
```typescript
{
  products: Product[];
  loading: boolean;
  error: string | null;
  createProduct: (product) => Promise;
  updateProduct: (id, updates) => Promise;
  deleteProduct: (id) => Promise;
  refetch: () => Promise;
}
```

---

### `src/lib/api.ts`
**Cliente HTTP para llamadas al backend:**
- `api.listProducts()` - GET /products
- `api.getProduct(id)` - GET /products/{id}
- `api.createProduct(product)` - POST /products
- `api.updateProduct(id, updates)` - PUT /products/{id}
- `api.deleteProduct(id)` - DELETE /products/{id}

**Obtiene API_URL de:**
1. `window.__ENV.VITE_API_URL` (runtime config)
2. `import.meta.env.VITE_API_URL` (build-time)
3. Fallback: `https://your-function-url.lambda-url.us-east-1.on.aws`

---

### `src/lib/types.ts`
**Tipos TypeScript:**
```typescript
interface Product {
  productId: string;
  name: string;
  description: string;
  price: number;
  category: string;
  stock: number;
  imageUrl: string;
  createdAt?: string;
  updatedAt?: string;
  
  // AI Fields (opcionales, agregados por sesiones)
  aiLabels?: string[];
  aiModeration?: Record<string, unknown>;
  aiAltText?: string;
  aiSentiment?: Record<string, unknown>;
  aiTranslations?: Record<string, { name: string; description: string }>;
  aiAudioUrl?: string;
  aiDescription?: string;
  aiEmbedding?: number[];
}
```

---

## 🔄 Flujo de Datos

### Al Cargar la Página:
```
App monta
  ↓
useProducts() ejecuta fetchProducts()
  ↓
api.listProducts() hace GET a /products
  ↓
DynamoDB retorna productos
  ↓
Se renderiza <ProductCard> x cada producto
  ↓
Cada ProductCard contiene <ProductAIFeatures>
  ↓
ProductAIFeatures carga window.__LAMBDA_URLS
  ↓
Usuario ve la página con todos los productos
```

### Al Hacer Clic en "Traducir":
```
ProductAIFeatures.handleTranslate()
  ↓
Extrae translateUrl de window.__LAMBDA_URLS
  ↓
Hace POST a https://...lambda-url.../？id={productId}
  ↓
Body: { target: "en" }
  ↓
Lambda de Translate traduce con Amazon Translate
  ↓
Devuelve { translation: { name, description } }
  ↓
ProductAIFeatures actualiza currentProduct
  ↓
Se muestra la traducción en la vista previa
```

### Al Hacer Clic en "Escuchar descripción":
```
ProductAIFeatures.handleGenerateAudio()
  ↓
Extrae voiceUrl de window.__LAMBDA_URLS
  ↓
Hace POST a https://...lambda-url.../？id={productId}
  ↓
Body: { lang: "es" o "en" }
  ↓
Lambda de Polly genera audio con Amazon Polly
  ↓
Sube a S3 con URL prefirmada (3600 segundos)
  ↓
Devuelve { audioUrl: "https://..." }
  ↓
audioRef.current.src = audioUrl
  ↓
Reproductor se llena con el audio
  ↓
Usuario presiona ▶ para escuchar
```

---

## 🎨 Estilos (Tailwind CSS)

### Colores principales:
- **Primario:** `bg-blue-600`, `text-blue-700`
- **Secundario:** `bg-green-100`, `text-green-700` (audio)
- **Error:** `bg-red-50`, `text-red-600`
- **Fondo:** `bg-gray-50`, `bg-white`

### Componentes reutilizables:
```typescript
// Botón primario
className="bg-blue-600 text-white hover:bg-blue-700"

// Botón secundario
className="bg-gray-100 text-gray-700 hover:bg-gray-200"

// Input
className="border border-gray-200 rounded-lg focus:ring-2 focus:ring-blue-500"

// Card
className="bg-white rounded-lg shadow-md overflow-hidden"
```

---

## 📦 Dependencias Principales

```json
{
  "react": "^18.x",
  "react-dom": "^18.x",
  "typescript": "^5.x",
  "tailwindcss": "^3.x",
  "lucide-react": "^0.x" // Iconos
}
```

---

## 🚀 URLs de Lambda (inyectadas en runtime)

```javascript
// frontend/dist/env-config.js
window.__LAMBDA_URLS = {
  translateUrl: 'https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/',
  voiceUrl: 'https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/'
};
```

---

## 📝 Configuración de Entorno

**Archivo:** `frontend/.env.example`
```
VITE_API_URL=https://wkhzf6meg52zmd6xyo2gvxne6m0ucyxk.lambda-url.us-east-1.on.aws/
```

**Inyección en Deploy:**
```bash
./scripts/inject-env.sh --api-url $API_URL --dist-dir frontend/dist
```

Esto genera `frontend/dist/env-config.js` con:
- `window.__ENV.VITE_API_URL`
- `window.__LAMBDA_URLS.translateUrl`
- `window.__LAMBDA_URLS.voiceUrl`
