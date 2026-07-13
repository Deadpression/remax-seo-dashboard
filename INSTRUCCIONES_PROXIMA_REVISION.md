# Instrucciones para la revisión SEO semanal — RE/MAX Chile

**Revisión actual:** Jun 2026 S1 (2026-06-23)  
**Frecuencia:** semanal  
**Dashboard:** https://deadpression.github.io/remax-seo-dashboard/  
**Repo:** https://github.com/Deadpression/remax-seo-dashboard

---

## Resumen de resultados actuales

| Métrica | Total | Deficientes | % | Umbral |
|---------|-------|-------------|---|--------|
| Agentes activos | 1.087 | 730 | 67% | < 600 car |
| Oficinas | 49 | 24 | 49% | < 600 car |
| Propiedades | 9.133 | 588 | 6,4% | < 590 car ó < 30 pal |

| Score Web | Valor |
|-----------|-------|
| SEO Audit | 52/100 |
| Technical SEO | 45/100 |
| AI Visibility | 48/100 |

---

## Umbrales de deficiencia

| Tipo | Mínimo palabras | Mínimo caracteres |
|------|-----------------|-------------------|
| Propiedad | 30 | 590 |
| Agente | — | 600 |
| Oficina | — | 600 |

---

## Reglas de exclusión (aplicar siempre)

- **Agentes inactivos:** excluir si `activo = "Inactivo"` en el Excel fuente (o `Terminated = true` vía REX API).
- **Agentes de la Training Office:** excluir SIEMPRE todos los agentes cuyo `OfficeID` sea el de la Training Office (`1028001`, shortlink `TRAININGOFFICE`). *Directiva permanente del usuario, agregada 2026-07-13.*
- **Perfiles "Administración":** excluir SIEMPRE todos los agentes cuyo nombre contenga la palabra "Administración" (son perfiles genéricos de oficina, no agentes reales). *Directiva permanente del usuario, agregada 2026-07-13.*
- **Propiedades no encontradas en API:** excluir si `words = -1` o `chars = -1` en el TSV.
- **Propiedades sin descripción:** excluir si `words = 0` y `chars = 0`.
- **Propiedades sin datos:** excluir si no aparecen en el mapa TSV.

---

## Lecciones aprendidas

### 1. Descripción real no siempre está en `ListingDescriptions[0]`

El API devuelve un array `ListingDescriptions` con múltiples entradas por propiedad. El scraper solo debe usar la **descripción más larga** entre todos los índices del array:

```javascript
const descs = (c?.ListingDescriptions||[]).map(ld=>sh(ld?.Description||'')).filter(Boolean);
const best  = descs.length ? descs.reduce((a,b)=>a.length>=b.length?a:b) : '';
```

### 2. La paginación del API es inestable

Al paginar con `skip:N` sobre 13.000+ registros, algunos pueden desaparecer entre páginas si el índice se actualiza. Todo deficiente debe verificarse individualmente antes de reportarse.

---

## Script del browser — scraping masivo de propiedades

Abrir Chrome en `https://www.remax.cl`, DevTools → Console, pegar y ejecutar:

```javascript
function sh(s){if(!s)return'';return s.replace(/<[^>]+>/g,'').replace(/&[a-z]+;/g,' ').replace(/&#\d+;/g,' ').replace(/\s+/g,' ').trim();}

window._lr = [];
window._lrDone = false;
window._lrProg = 0;

(async()=>{
  const PS = 50;
  let skip = 0;
  const r0 = await fetch('/search/listing-search/docs/search', {
    method:'POST', headers:{'Content-Type':'application/json'},
    body: JSON.stringify({search:'*', filter:'content/MacroRegionId eq 1028', top:1, count:true})
  });
  const d0 = await r0.json();
  const total = d0['@odata.count'] || 14000;

  while(skip < total) {
    try {
      const r = await fetch('/search/listing-search/docs/search', {
        method:'POST', headers:{'Content-Type':'application/json'},
        body: JSON.stringify({search:'*', filter:'content/MacroRegionId eq 1028', top:PS, skip})
      });
      const d = await r.json();
      if(!d.value?.length) break;
      for(const v of d.value) {
        const c = v.content;
        const descs = (c?.ListingDescriptions||[]).map(ld=>sh(ld?.Description||'')).filter(Boolean);
        const best  = descs.length ? descs.reduce((a,b)=>a.length>=b.length?a:b) : '';
        window._lr.push({
          mls_id: c?.MLSID||'',
          statusUID: c?.ListingStatusUID,
          dp: best ? best.split(/\s+/).filter(Boolean).length : 0,
          dc: best.length
        });
      }
      skip += PS;
      window._lrProg = skip;
    } catch(e) { await new Promise(r=>setTimeout(r,500)); }
    await new Promise(r=>setTimeout(r,60));
  }
  window._lrDone = true;
  console.log('DONE:', window._lr.length, 'registros');
})();
```

Monitorear con `window._lrProg` — llega a ~14.000 en ~4 minutos.

---

## Verificación de deficientes (paso obligatorio)

Verificar cada propiedad que aparezca deficiente con consulta directa:

```javascript
function sh(s){if(!s)return'';return s.replace(/<[^>]+>/g,'').replace(/&[a-z]+;/g,' ').replace(/&#\d+;/g,' ').replace(/\s+/g,' ').trim();}
const id = 'REEMPLAZAR-CON-MLS-ID';
const r  = await fetch('/search/listing-search/docs/search', {
  method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({search:`"${id}"`, top:5})
});
const d = await r.json();
const c = d.value?.find(v=>v.content?.MLSID===id)?.content;
const descs = (c?.ListingDescriptions||[]).map((ld,i)=>({
  i, w: sh(ld?.Description||'').split(/\s+/).filter(Boolean).length,
  preview: sh(ld?.Description||'').substring(0,100)
}));
console.log(JSON.stringify({mlsId:c?.MLSID, statusUID:c?.ListingStatusUID, descs}));
```

Si tiene descripción real en algún índice → no es deficiente, corregir el dato.

---

## Archivos de referencia

| Archivo | Descripción |
|---------|-------------|
| `ilist_resultados_YYYY-MM-DD.xlsx` | Excel entregable con Agentes, Oficinas y Propiedades |
| `seo_dashboard_data/dashboard_updated.html` | Dashboard local con datos + histórico |
| `seo_dashboard_data/index.html` | Copia para publicar en GitHub Pages |
| `seo_dashboard_data/add_snapshot.py` | Script para agregar snapshot al dashboard |
| `seo_dashboard_data/robots.txt` | Evita indexación del dashboard en buscadores |

---

## Proceso completo de revisión semanal

> Claude ejecuta **todo** este proceso. El usuario solo necesita: (1) subir los 3 Excel de iList al inicio, (2) correr el script del browser en remax.cl, y (3) aprobar el push final a GitHub.

---

### PARTE A — Scores Web

Claude ejecuta las tres skills sobre `https://www.remax.cl` con Chrome conectado:

1. `searchfit-seo:seo-audit` → anotar score /100
2. `searchfit-seo:technical-seo` → anotar score /100
3. `searchfit-seo:ai-visibility` → anotar score /100

---

### PARTE B — iList (agentes, oficinas, propiedades)

4. Usuario descarga desde el portal iList: `OurAgents.xlsx`, `OurOffices.xlsx`, `OurProperties.xlsx`
5. Usuario sube los 3 archivos a Claude en Cowork
6. Claude lee los Excel y extrae datos de agentes y oficinas
7. Usuario ejecuta el script del browser en `remax.cl` y espera `window._lrDone === true`
8. Claude procesa los resultados del scraping con las reglas de exclusión y umbrales
9. Claude verifica individualmente cada propiedad con < 30 palabras o < 590 caracteres
10. Claude genera `ilist_resultados_YYYY-MM-DD.xlsx`:
    - Hoja **Agentes**: excluye inactivos, marca deficientes (< 600 car) en rojo
    - Hoja **Oficinas**: marca deficientes (< 600 car) en rojo
    - Hoja **Propiedades**: excluye -1/-1 y 0/0, marca deficientes (< 590 car ó < 30 pal) en rojo
    - En todas las hojas: deficientes primero, OK después; fila 1 congelada

---

### PARTE C — Actualizar dashboard y publicar

11. Claude actualiza `dashboard_updated.html` con scores Web e iList del período
12. Claude ejecuta el snapshot automáticamente:

    ```bash
    cd "seo_dashboard_data"
    python add_snapshot.py --json '{
      "fecha": "YYYY-MM-DD",
      "label": "Mes YYYY SN",
      "ilist": {
        "agentes":     {"total": N, "def": N},
        "oficinas":    {"total": N, "def": N},
        "propiedades": {"total": N, "def": N}
      },
      "web": {"seo_audit": N, "technical_seo": N, "ai_visibility": N},
      "gsc": {"clicks": N, "impressions": N, "ctr": N, "position": N}
    }'
    ```

    > Etiqueta: `Jun 2026 S1`, `Jun 2026 S2`, `Jul 2026 S1`, etc.

13. Claude copia `dashboard_updated.html` → `index.html` y sube ambos archivos a GitHub:
    - Ir a https://github.com/Deadpression/remax-seo-dashboard
    - `Add file → Upload files` → subir `index.html` y `robots.txt`
    - Commit a `main` con mensaje: `Revisión YYYY-MM-DD`
    - GitHub Pages actualiza en ~1