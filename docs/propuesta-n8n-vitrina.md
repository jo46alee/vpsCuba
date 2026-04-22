# Propuesta técnica: n8n + scraping + vitrina e-commerce (TrendShopX2025)

## 1) Resumen ejecutivo

Se propone una arquitectura **ETL + catálogo desacoplado**:

- **Extracción** automática desde `https://trendshopx2025.biznecubano.com/` con n8n.
- **Normalización** de datos (nombre, precio, moneda, stock, imagen, categoría, URL, hash de cambios).
- **Persistencia** en una base estructurada (recomendado: Supabase).
- **Frontend** separado (recomendado: Next.js) con estilo visual inspirado en VirtuMall.
- **Sincronización periódica** (cada 30-60 minutos) con estrategia anti-errores.

### Hallazgo técnico clave del sitio origen

En el HTML principal aparecen múltiples textos de marcador como **"Cargando información, espere..."** en listados de productos, lo cual sugiere que parte del contenido (especialmente precios) se inyecta por JavaScript o por llamadas asíncronas posteriores al render inicial.

Por eso el diseño incluye dos métodos de extracción:
1. **Método A (preferido):** endpoint/API subyacente si existe.
2. **Método B (fallback):** navegador headless (Playwright/Puppeteer) cuando el precio no esté en HTML.

## 2) Arquitectura recomendada

### 2.1 Componentes

1. **n8n (orquestación):**
   - Programa ejecuciones.
   - Descubre URLs de producto.
   - Extrae campos.
   - Normaliza y valida.
   - Upsert de catálogo.
   - Emite JSON para frontend/caché.

2. **Capa de datos (recomendada: Supabase Postgres):**
   - Tabla `products` (catálogo normalizado).
   - Tabla `sync_runs` (auditoría de ejecuciones).
   - Índices por `sku/external_id`, `slug`, `updated_at`.

3. **Frontend (recomendado: Next.js + Tailwind):**
   - Landing estilo VirtuMall (hero + trust badges + categorías + grid).
   - Consumo directo de Supabase (SSR/ISR) o de `products.json` publicado por n8n.
   - CTA de compra que redirige al producto original.

4. **Hosting/CDN:**
   - Vercel/Netlify para frontend.
   - n8n self-hosted o n8n cloud.

### 2.2 Flujo de datos

`Schedule Trigger` → `Crawler` (lista productos) → `Extractor` (detalle por producto) → `Normalizer` → `Deduplicador` → `Upsert DB` → `Generación products.json` → `Webhook build/revalidate frontend`.

### 2.3 Actualización de precios

- Ejecución incremental cada 30-60 min.
- `hash_payload` para detectar cambios reales.
- Si cambia precio/stock: actualizar `updated_at` y registrar snapshot.

### 2.4 Manejo de precios dinámicos

- Intento 1: localizar endpoint interno (XHR/JSON) y consumirlo directamente.
- Intento 2: si no existe endpoint usable, usar Playwright para esperar selector de precio y extraer DOM renderizado.
- Recomendación de estabilidad: **endpoint interno > headless** (más rápido, barato y menos frágil).

## 3) Workflow n8n paso a paso

### 3.1 Flujo principal (recomendado)

1. **`Schedule Trigger - Cada 30m`**
   - Frecuencia: cada 30 minutos.

2. **`HTTP Request - Home HTML`**
   - GET `https://trendshopx2025.biznecubano.com/`
   - `Response Format`: String.
   - Header `User-Agent`: navegador real.

3. **`HTML Extract - Links de productos`**
   - Selector ejemplo: `a[href*="/p/"]`
   - Extraer `href`.

4. **`Code - Normalizar URLs y dedupe inicial`**
   - Convierte a URL absoluta.
   - Quita repetidos por URL.

5. **`Split In Batches - Productos`**
   - Tamaño 10-20 por lote para no sobrecargar.

6. **`HTTP Request - Producto HTML`**
   - GET `{{$json.url}}`

7. **`HTML Extract - Campos básicos`**
   - `name`, `image`, `availability`, `category breadcrumbs`.
   - Si `price` no existe, marcar `needs_dynamic=true`.

8. **`IF - ¿Falta precio?`**
   - Condición: `{{$json.price === null || $json.price === undefined}}`

9A. **Rama A (precio visible): `Code - Parse Price`**
   - Parsea moneda/valor y stock.

9B. **Rama B (precio dinámico): `Execute Command / Webhook a microservicio`**
   - Invoca Playwright (servicio externo o script) para extraer precio renderizado.

10. **`Merge - Unificar A/B`**

11. **`Set - Modelo final`**
   - Campos normalizados: `sku, name, price, currency, stock, category, image, url, updated_at, hash_payload`.

12. **`IF - Validación mínima`**
   - `name && url && price != null`.
   - Si falla, enviar a `Error Queue`.

13. **`Supabase - Upsert products`**
   - Clave única: `url` o `external_id`.

14. **`Postgres/Supabase - Marcar inactivos`**
   - Productos no vistos en corrida actual → `is_active=false`.

15. **`HTTP Request - Publicar JSON` (opcional)**
   - Subir `products.json` a storage bucket/CDN.

16. **`Webhook - Revalidate Frontend`**
   - Llama endpoint de revalidación ISR de Next.js.

17. **`Slack/Telegram - Reporte`**
   - Totales: nuevos, actualizados, desactivados, errores.

### 3.2 Expresiones n8n útiles

- URL absoluta:
  - `{{ $json.href.startsWith('http') ? $json.href : 'https://trendshopx2025.biznecubano.com' + $json.href }}`

- Parse número de precio:
  - `{{ Number(($json.price_raw || '').replace(/[^\d.,]/g,'').replace('.', '').replace(',', '.')) }}`

- Moneda por símbolo:
  - `{{ ($json.price_raw || '').includes('USD') ? 'USD' : (($json.price_raw || '').includes('CUP') ? 'CUP' : 'CUP') }}`

## 4) Estrategia de extracción (2 alternativas)

## 4.1 Alternativa 1: HTML + endpoint/API (recomendada)

**Cómo:**
- Descargar HTML inicial.
- Detectar scripts/XHR del sitio.
- Reproducir petición JSON de productos/precios con `HTTP Request`.

**Ventajas:**
- Más estable, rápida y barata.
- Menor consumo de CPU.
- Más fácil de escalar.

**Riesgos:**
- Endpoint no documentado puede cambiar.

## 4.2 Alternativa 2: Headless browser

**Cómo:**
- Playwright abre página de producto.
- Espera selector de precio (`waitForSelector`).
- Extrae texto renderizado del DOM.

**Ventajas:**
- Funciona incluso con render 100% JS.

**Desventajas:**
- Más costo de infraestructura.
- Más frágil ante cambios visuales.
- Más lento.

## 4.3 Recomendación final

Implementa **pipeline híbrido con fallback**:
- Primero endpoint/API.
- Si falla o no devuelve precio, usar headless solo para esos productos.

## 5) Estructura de datos recomendada

```json
{
  "sku": "",
  "external_id": "",
  "name": "",
  "slug": "",
  "description": "",
  "price": 0,
  "currency": "CUP",
  "stock": 0,
  "availability": "in_stock",
  "category": "",
  "image": "",
  "url": "",
  "source": "trendshopx2025.biznecubano.com",
  "hash_payload": "",
  "is_active": true,
  "updated_at": "2026-04-22T00:00:00.000Z"
}
```

## 6) Código útil (n8n Code nodes)

### 6.1 Extraer links de productos desde home

```js
const base = 'https://trendshopx2025.biznecubano.com';
const html = $json.html || '';
const regex = /href=["']([^"']*\/p\/[a-zA-Z0-9_-]+)["']/g;
const urls = new Set();
let m;
while ((m = regex.exec(html)) !== null) {
  const href = m[1];
  const abs = href.startsWith('http') ? href : `${base}${href}`;
  urls.add(abs.split('?')[0]);
}
return [...urls].map(url => ({ json: { url } }));
```

### 6.2 Limpiar nombre, precio y stock

```js
function cleanText(v='') {
  return v.replace(/\s+/g, ' ').trim();
}

function parsePrice(raw='') {
  const txt = cleanText(raw).toUpperCase();
  const currency = txt.includes('USD') || txt.includes('$') ? 'USD' : (txt.includes('CUP') ? 'CUP' : 'CUP');
  const n = txt.replace(/[^\d.,]/g, '').replace(/\.(?=\d{3}(\D|$))/g, '').replace(',', '.');
  const price = Number(n);
  return Number.isFinite(price) ? { price, currency } : { price: null, currency };
}

function parseStock(raw='') {
  const txt = cleanText(raw).toLowerCase();
  if (txt.includes('solo quedan')) {
    const n = Number((txt.match(/(\d+)/) || [])[1] || 0);
    return n;
  }
  if (txt.includes('disponible')) return 999;
  const n = Number((txt.match(/(\d+)/) || [])[1] || 0);
  return Number.isFinite(n) ? n : 0;
}

const out = items.map(item => {
  const j = item.json;
  const pp = parsePrice(j.price_raw || '');
  return {
    json: {
      ...j,
      name: cleanText(j.name || ''),
      price: pp.price,
      currency: pp.currency,
      stock: parseStock(j.stock_raw || ''),
    }
  };
});
return out;
```

### 6.3 Transformar al formato final

```js
const crypto = require('crypto');

function slugify(s='') {
  return s.toLowerCase().normalize('NFD').replace(/[\u0300-\u036f]/g, '')
    .replace(/[^a-z0-9]+/g, '-').replace(/(^-|-$)/g, '');
}

return items.map(item => {
  const p = item.json;
  const payload = {
    sku: p.sku || '',
    external_id: p.external_id || p.url.split('/').pop(),
    name: p.name,
    slug: slugify(p.name),
    description: p.description || '',
    price: p.price,
    currency: p.currency || 'CUP',
    stock: Number.isFinite(p.stock) ? p.stock : 0,
    availability: (p.stock ?? 0) > 0 ? 'in_stock' : 'out_of_stock',
    category: p.category || 'Sin categoría',
    image: p.image || '',
    url: p.url,
    source: 'trendshopx2025.biznecubano.com',
    is_active: true,
    updated_at: new Date().toISOString(),
  };
  payload.hash_payload = crypto.createHash('sha1').update(JSON.stringify({
    name: payload.name,
    price: payload.price,
    currency: payload.currency,
    stock: payload.stock,
    image: payload.image,
    category: payload.category,
  })).digest('hex');

  return { json: payload };
});
```

### 6.4 Evitar duplicados (por URL o external_id)

```js
const map = new Map();
for (const item of items) {
  const p = item.json;
  const key = p.external_id || p.url;
  if (!map.has(key)) map.set(key, item);
}
return [...map.values()];
```

### 6.5 Actualizar existentes (upsert lógico)

```js
// Supone que llega item con existing (desde DB) y incoming (nuevo)
return items.map(({ json }) => {
  const existing = json.existing || null;
  const incoming = json.incoming;

  if (!existing) {
    return { json: { ...incoming, op: 'insert' } };
  }

  if (existing.hash_payload !== incoming.hash_payload) {
    return { json: { ...incoming, id: existing.id, op: 'update' } };
  }

  return { json: { ...existing, op: 'skip' } };
});
```

## 7) Frontend estilo VirtuMall (adaptado)

## 7.1 Estructura recomendada

1. Header (logo + CTA + navegación)
2. Hero elegante (mensaje + botón)
3. Trust badges (entrega rápida, soporte, pago seguro)
4. Categorías destacadas (chips/cards)
5. Grid de productos (cards modernas)
6. Sección “cómo comprar”
7. Footer con contacto

## 7.2 Estilo visual

- Fondo claro + gradientes suaves.
- Tipografía limpia (Inter/Poppins).
- Cards con bordes redondeados y sombra ligera.
- Botones primarios con alto contraste.
- Etiquetas (badge) para stock/ofertas.

## 7.3 Responsive

- Móvil: 1 columna.
- Tablet: 2 columnas.
- Desktop: 3-4 columnas.

## 7.4 Componente de tarjeta

- Imagen 1:1
- Nombre en 2 líneas
- Precio destacado
- Disponibilidad
- Botón “Comprar ahora” → URL origen

## 8) Recomendación de publicación

### Comparativa rápida

1. **n8n + Google Sheets + Softr**
   - Muy rápido de lanzar.
   - Limitado para personalización avanzada.

2. **n8n + Supabase + Next.js** ✅ **Recomendada**
   - Mejor equilibrio: costo, escalabilidad, control visual.
   - Upsert robusto y frontend profesional.

3. **n8n + JSON + Netlify/Vercel**
   - Barato y simple.
   - Menos flexible para filtros complejos y panel admin.

4. **n8n + WordPress**
   - Rápido con plugins.
   - Mayor mantenimiento (plugins/seguridad).

### Elección final

Para tu caso (actualizaciones periódicas + diseño moderno + control total):
**n8n + Supabase + Next.js**.

## 9) Problemas probables y soluciones

1. **Precio no aparece en HTML**
   - Solución: fallback a Playwright y/o endpoint interno.

2. **Bloqueo por rate limit**
   - Solución: lotes pequeños + retries exponenciales + User-Agent.

3. **Cambios en estructura HTML**
   - Solución: parseo por endpoint JSON prioritario + tests de selectors.

4. **Duplicados por URLs con parámetros**
   - Solución: canonicalizar URL (`split('?')[0]`).

5. **Productos eliminados en origen**
   - Solución: marcado `is_active=false` si no aparece en corrida.

6. **Frontend con datos stale**
   - Solución: webhook de revalidación ISR tras sincronizar.
