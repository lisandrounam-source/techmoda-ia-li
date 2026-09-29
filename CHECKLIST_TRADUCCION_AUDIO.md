# ✅ Checklist: Traducción y Audio Funcionando

## 🔍 Paso 1: Verificar que el Frontend Esté Actualizado

### A. Abre DevTools (F12) y ve a la consola

```javascript
// En la consola del navegador, verifica que esto exista:
console.log(window.__LAMBDA_URLS);
// Debería mostrar:
// { 
//   translateUrl: 'https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/',
//   voiceUrl: 'https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/'
// }
```

**Si aparece `undefined` o `{}`:**
- Presiona **Ctrl+F5** (o Cmd+Shift+R)
- El navegador tiene caché viejo

---

## 🌐 Paso 2: Verificar que el Backend Responde

```bash
# En terminal:

# 1. Obtén un producto
API_URL="https://wkhzf6meg52zmd6xyo2gvxne6m0ucyxk.lambda-url.us-east-1.on.aws/"
PRODUCT_ID=$(curl -s "${API_URL%/}/products" | python3 -c "import sys, json; print(json.load(sys.stdin)['products'][0]['productId'])")

echo "Product ID: $PRODUCT_ID"

# 2. Prueba Translate
curl -s -X POST "https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/?id=${PRODUCT_ID}" \
  -H "Content-Type: application/json" \
  -d '{"target":"en"}' | python3 -m json.tool

# Deberías ver:
# {
#   "productId": "...",
#   "target": "en",
#   "translation": {
#     "name": "...",
#     "description": "..."
#   }
# }

# 3. Prueba Voice
curl -s -X POST "https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/?id=${PRODUCT_ID}" \
  -H "Content-Type: application/json" \
  -d '{"lang":"es"}' | python3 -m json.tool

# Deberías ver:
# {
#   "productId": "...",
#   "lang": "es",
#   "voice": "Lupe",
#   "audioUrl": "https://...s3...mp3?X-Amz-...",
#   "expiresIn": 3600
# }
```

---

## 🎯 Paso 3: Probar en la Página

### A. Abre la página
```
https://d2eiybqber8s8k.cloudfront.net
```

### B. En cada tarjeta de producto deberías ver:

```
┌─────────────────┐
│ [Imagen]        │
├─────────────────┤
│ Nombre          │
│ Descripción     │
│ Precio          │
│                 │
│ [ES EN] [Traducir]   ← DEBERÍAS VER ESTO
│                 │
│ Preview texto   │
│                 │
│ 🔊 Escuchar     ← Y ESTO
│ [Audio player]  │
│                 │
│ [Agregar...]    │
└─────────────────┘
```

---

## 🧪 Paso 4: Probar Traducción

### ✅ Versión que Funciona:

1. **Haz clic en EN**
   - El botón debe cambiar a fondo blanco con azul

2. **Haz clic en "Traducir"**
   - El botón debe mostrar un spinner 🔄
   - Espera 2-3 segundos
   - El texto debajo debe cambiar al inglés
   - El spinner desaparece

3. **Sin errores** → La traducción funcionó ✅

### ❌ Si ves error "Failed to fetch":

1. Abre DevTools: **F12**
2. Pestaña **Console**
3. Busca el error rojo
4. Copia el error exacto
5. Corre esto en la consola:

```javascript
// Verifica que las URLs están cargadas
console.log('Translate URL:', window.__LAMBDA_URLS?.translateUrl);
console.log('Voice URL:', window.__LAMBDA_URLS?.voiceUrl);

// Intenta una petición manualmente
fetch('https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/?id=PRODUCT_ID', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ target: 'en' })
}).then(r => r.json()).then(console.log).catch(console.error);
```

---

## 🔊 Paso 5: Probar Audio

### ✅ Versión que Funciona:

1. **Haz clic en "Escuchar descripción"**
   - El botón debe mostrar "Generando audio..." 🔄
   - Espera 2-3 segundos
   - Aparece un reproductor de audio debajo con `▶ ⏸ ⏹`
   - El reproductor se llena con la barra de tiempo

2. **Haz clic en ▶**
   - Escuchas la descripción en voz
   - Sin errores → Audio funcionó ✅

### ❌ Si el reproductor está vacío:

1. Abre DevTools: **F12**
2. Pestaña **Network**
3. Haz clic en "Escuchar descripción"
4. Busca una petición a la URL de Voice
5. Verifica que devuelva **200 OK**
6. Si devuelve **404** o **403**, el problema es de permisos

---

## 🔧 Paso 6: Si Nada Funciona - Redeploy

```bash
cd /workshop/capstone

# 1. Ejecuta el script automático
bash scripts/deploy-full.sh

# Espera 2-3 minutos

# 2. Abre la página
# https://d2eiybqber8s8k.cloudfront.net

# 3. Presiona Ctrl+F5 (o Cmd+Shift+R)

# 4. Intenta traducir/audio nuevamente
```

---

## 📋 Checklist Final

- [ ] Veo los botones ES/EN en cada producto
- [ ] Veo el botón "Traducir"
- [ ] Hago clic EN → el botón se activa
- [ ] Hago clic Traducir → el texto cambia al inglés (sin error)
- [ ] Veo el botón "Escuchar descripción"
- [ ] Hago clic → aparece reproductor de audio
- [ ] Hago clic ▶ → escucho la descripción en voz

---

## 🛠️ Mantener Cambios sin Perder en Futuros Deploys

### Pasos para desarrollar sin perder ProductAIFeatures:

```bash
cd /workshop/capstone

# 1. Hacer cambios en los archivos:
# - frontend/src/components/ProductAIFeatures.tsx
# - frontend/src/components/ProductCard.tsx
# - Cualquier archivo en frontend/src/

# 2. Commitear si es trabajo importante
git add frontend/src/components/
git commit -m "Add translation and audio features"

# 3. Deploy (preserva automáticamente)
bash scripts/deploy-full.sh

# 4. Verificar en la página
# https://d2eiybqber8s8k.cloudfront.net
```

### El script `deploy-full.sh` hace:
- ✅ No elimina archivos (usa `aws s3 sync` sin `--delete`)
- ✅ Inyecta URLs de Lambda automáticamente
- ✅ Limpia caché de CloudFront
- ✅ Preserva imágenes en `assets/` e `img/`

---

## 🚨 Problema Común: Caché Viejo

**Síntoma:** Ves los botones pero no funcionan

**Causa:** El navegador tiene caché viejo

**Solución:**
```
1. Presiona Ctrl+F5 (Windows/Linux)
2. O Cmd+Shift+R (Mac)
3. O abre DevTools → Pestaña Application → Limpiar Storage
```

---

## 📞 Resumen de URLs Importantes

```bash
# Frontend (lo que ves)
https://d2eiybqber8s8k.cloudfront.net

# Backend (productos)
https://wkhzf6meg52zmd6xyo2gvxne6m0ucyxk.lambda-url.us-east-1.on.aws/products

# Traducción
https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/?id={ID}&target=en

# Audio
https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/?id={ID}&lang=es
```

---

## ✨ Verificación Rápida (1 minuto)

```bash
# Corre esto para verificar que todo está listo:

echo "1. ¿Frontend está actualizado?"
curl -s https://d2eiybqber8s8k.cloudfront.net | grep -q "ProductAIFeatures\|Traducir\|Escuchar" && echo "✅ SÍ" || echo "❌ NO"

echo ""
echo "2. ¿Backend responde?"
curl -s https://wkhzf6meg52zmd6xyo2gvxne6m0ucyxk.lambda-url.us-east-1.on.aws/products | python3 -c "import sys, json; d=json.load(sys.stdin); print(f'✅ {len(d[\"products\"])} productos' if 'products' in d else '❌ Error')"

echo ""
echo "3. ¿Translate está disponible?"
curl -s https://o4ibqyd4mzec3txtp5bcfaka5u0mbttw.lambda-url.us-east-1.on.aws/?id=test 2>&1 | grep -q "productId\|error" && echo "✅ SÍ" || echo "❌ NO"

echo ""
echo "4. ¿Voice está disponible?"
curl -s https://k5jc74guchouyoaph4rv4wy4s40oaacb.lambda-url.us-east-1.on.aws/?id=test 2>&1 | grep -q "productId\|error" && echo "✅ SÍ" || echo "❌ NO"
```
