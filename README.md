# Expediente C

Herramienta web para armar un instrumento ambiental de **Categoría C** del Ministerio de Ambiente y Recursos Naturales de Guatemala — Evaluación Ambiental Inicial (EAI) o Diagnóstico Ambiental de Bajo Impacto (DABI).

Guía a la persona por siete pasos en lenguaje llano, le dice qué instrumento le corresponde, y le arma la lista de documentos que su proyecto va a necesitar antes de que llegue a ventanilla.

**En línea:** https://arqcepollina-art.github.io/expediente-c

---

## Qué hace

- **Determina el instrumento** a partir de una sola pregunta: si el proyecto ya está construido o todavía no.
- **Clasifica contra el Listado Taxativo** (Acuerdo Ministerial 402-2021), con las 752 actividades embebidas. Se busca **por descripción** —escribiendo «taller», «electrodomésticos», «bodega»— o eligiendo sector y subsector. Las filas ya cotejadas contra el texto del Acuerdo se marcan como verificadas; las demás muestran sus umbrales con la advertencia de que provienen de un OCR y pueden venir cruzados.
- **Anticipa los compromisos que impone el MARN.** Según la actividad elegida, muestra lo que la autoridad resolvió en expedientes reales: el bloque universal, el de fase operativa, el de fase constructiva y los propios de cada tipología. Un compromiso que no se propuso en el instrumento se impone igual, y para entonces el proponente ya no puede presupuestarlo.
- **Se adapta a la vía de presentación.** En ventanilla del MARN muestra unos campos; en SAGA muestra los que esa plataforma pide de más (CUI, NIT y contacto del representante, fase de abandono, combustibles, lubricantes, clasificación de residuos).
- **Advierte lo que el proyecto dispara.** Aguas residuales especiales piden Ingeniero Sanitarista; el SIGAP pide CONAP; movimiento de tierra pide plano de curvas de nivel. Las alertas se actualizan conforme se llena.
- **Arma la lista de documentos** — legales y técnicos — según las respuestas.
- **Exporta e importa** el expediente como archivo `.json` para seguirlo después o pasarlo a un consultor.

## Cómo está hecho

Un solo archivo, `index.html`. Sin dependencias, sin build, sin backend. Todo el JavaScript y el CSS van adentro, y el Listado Taxativo va embebido como JSON. La única solicitud externa es la de Google Fonts; si no carga, la tipografía cae en la del sistema y la herramienta funciona igual.

Los datos que escribe la persona se guardan en `localStorage` — solo en su navegador. No hay servidor, no hay base de datos, nadie más los ve.

```
index.html     la aplicación completa
.nojekyll      evita que GitHub Pages procese el sitio con Jekyll
.gitignore
README.md
```

## Publicarlo en GitHub Pages

1. Crear un repositorio nuevo llamado `expediente-c`.
2. Subir estos archivos a la rama `main`.
3. En el repositorio: **Settings → Pages**.
4. En *Source* elegir **Deploy from a branch**; en *Branch* elegir `main` y carpeta `/ (root)`. Guardar.
5. A los pocos minutos el sitio queda en `https://<usuario>.github.io/expediente-c`.

Para actualizarlo basta con reemplazar `index.html` y hacer commit: GitHub vuelve a publicar solo.

## Actualizar el Listado Taxativo

El catálogo está embebido dentro de `index.html`, en la línea que empieza con `const TAX =`. Es un arreglo JSON donde cada actividad tiene esta forma:

```json
{
  "id":  "10-C-003",
  "sec": "INFRAESTRUCTURA, CONSTRUCCIÓN Y VIVIENDA",
  "sub": "VIVIENDA",
  "act": "Construcción de edificios.",
  "des": "Viviendas unifamiliares o bifamiliares (incluye muro de contención).",
  "fac": "Área a intervenir",
  "uni": "Metros cuadrados",
  "c":   [">500", ""],
  "cpga":["", ""],
  "b2":  ["", ""],
  "b1":  ["", ""],
  "a":   ["", ""],
  "umb": true,
  "catmin": "C"
}
```

Cada categoría lleva un par `[mínimo, máximo]`. Si el MARN reforma el Listado Taxativo, se reemplaza ese arreglo.

### Filas verificadas

El catálogo embebido proviene de un OCR del PDF oficial y arrastra dos defectos: umbrales cruzados o invertidos, y la actividad económica rellenada hacia abajo en subsectores enteros (unas 200 de las 752 filas). Por eso la búsqueda va por descripción y no por actividad.

Cuando una fila se coteja contra el texto del Acuerdo, se corrige en la constante `VERIFICADAS`, cerca de `taxSel()`:

```js
const VERIFICADAS = {
  '09-B-013': { act:'…', ciiu:'4759', c:['','<=500'], cpga:['>500',''], catmin:'C' }
};
```

La interfaz marca esas filas como verificadas y cambia la advertencia de umbrales. Ir agregando las filas que se usan a diario es más barato que limpiar las 752 de una vez, y es el camino para activar categorización automática sin arriesgar un veredicto equivocado.

## Limitaciones conocidas

- **Los umbrales numéricos son de referencia.** El catálogo se extrajo de una versión en hoja de cálculo del Acuerdo Ministerial 402-2021 que a su vez proviene de un OCR del PDF, y algunas filas traen los rangos cruzados o incompletos. Por eso la herramienta **no categoriza automáticamente**: muestra los umbrales y advierte que deben verificarse contra el texto oficial.
- No genera todavía el formulario en Word ni el anexo fotográfico.
- No cubre Categoría C con Plan de Gestión Ambiental. En SAGA el PGA se adjunta al mismo expediente, así que se puede agregar como módulo más adelante.

## Aviso

Esta herramienta ayuda a ordenar información y a anticipar requisitos. **No sustituye el criterio de un consultor ambiental** ni garantiza la aprobación de ningún expediente. La categorización final depende del Listado Taxativo vigente y de la resolución del Ministerio de Ambiente y Recursos Naturales.

## Base normativa

- Acuerdo Gubernativo 137-2016 — Reglamento de Evaluación, Control y Seguimiento Ambiental
- Acuerdo Ministerial 402-2021 — Listado Taxativo de Proyectos, Obras, Industrias o Actividades
- Acuerdo Gubernativo 236-2006 — Descargas y reúso de aguas residuales y disposición de lodos
- Acuerdo Gubernativo 164-2021 — Gestión integral de residuos y desechos sólidos
