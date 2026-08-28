# Notas de desarrollo

Contexto para retomar el proyecto. Registra por qué las cosas están como están, no solo qué hacen.

Última actualización: agosto 2026 · segunda versión: catálogo corregido, búsqueda por descripción y compromisos ambientales.

---

## Qué es y para quién

Herramienta para que una persona **arme su propio instrumento ambiental Categoría C** del MARN de Guatemala, y de paso entienda en qué se está metiendo antes de llegar a ventanilla.

El alcance cambió durante el desarrollo: arrancó como herramienta interna de una firma de consultoría y terminó apuntando a un público más amplio. Por eso el lenguaje es llano y hay explicaciones donde un consultor no las necesitaría. **Queda una decisión abierta:** si alguien sin criterio técnico llena esto y lo presenta, la responsabilidad profesional del instrumento queda en el aire. Vale considerar si el producto final es «armá tu borrador y un consultor lo revisa» en vez de «presentalo vos».

## Decisiones de arquitectura

**Un solo archivo, sin dependencias, sin backend.** Todo vive en `index.html`. No hay build ni framework. El motivo es longevidad: esto tiene que seguir funcionando dentro de cinco años sin que nadie actualice dependencias. La única solicitud externa es Google Fonts, y si falla la tipografía cae en la del sistema sin romper nada.

**El modelo de datos tiene la forma de SAGA, no la del formulario físico.** Esto es lo más importante del diseño. SAGA pide *más* campos y *más granulares* que el papel: dirección desglosada en calle/avenida/zona/departamento/municipio, fase de abandono, uso de combustibles, uso de lubricantes, clasificación de residuos, y NIT/correo/teléfono/cargo del representante legal. Si se captura con el modelo del papel, después no se puede llenar SAGA sin volver a llamar al cliente. Al revés sí funciona: del modelo de SAGA se deriva el texto corrido que pide el formulario físico.

**El campo `via` (`fisico` | `saga`) gobierna qué se muestra.** Los campos exclusivos de SAGA aparecen y desaparecen según ese valor.

**El instrumento se deduce de una sola pregunta.** `estado` = `sinEjecutar` → EAI (predictivo, futuro, Q.100). `estado` = `operando` → DABI (correctivo, presente, Q.150). Eso apaga las secciones de construcción, cambio de uso del suelo y geomorfología, que el propio formulario marca como «aplicable únicamente a instrumentos predictivos».

**La herramienta no categoriza automáticamente. Es deliberado.** Ver «Deuda conocida».

## El motor de coherencia

El hallazgo que originó el diseño: en los expedientes reales analizados, los errores no venían de desconocimiento técnico sino de **datos escritos más de una vez**. Un expediente traía «Fase II» en el formulario y «Fase III» en el anexo fotográfico; otro traía el NIT del representante legal donde iba el de la sociedad.

Hay nueve datos que se escriben una vez y deben aparecer idénticos en varios lugares: nombre del proyecto, dirección, coordenadas, las tres áreas, los volúmenes de corte/relleno/excavación, personal, horario, finca-folio-libro y licencia ambiental madre.

**La plataforma no es un formulario bonito: es un motor de coherencia.** Cualquier cambio futuro debería reforzar eso, no diluirlo.

## Reglas que disparan requisitos

Están en `alertas()` y `documentos()`:

| Condición | Consecuencia |
|---|---|
| `aguaEspecial === 'si'` | Memoria de cálculo y manual firmados por Ingeniero Sanitarista, en original |
| `tratamiento === 'si'` | Además, plano de detalles del sistema |
| `sigap === 'si'` | Opinión favorable y contrato del CONAP, antes de ingresar |
| `movTierra === 'si'` | Plano de curvas de nivel naturales y modificadas |
| `corteArboles === 'si'` | Volumen de madera obligatorio; posible gestión ante INAB |
| `cambioUso === 'si'` (solo EAI) | Plano de uso actual del suelo |
| `patrimonio === 'si'` | Posible dictamen del IDAEH |

Los dos primeros son los que más encarecen y atrasan un expediente. Conviene detectarlos en la primera reunión con el cliente, no a medio trabajo.

## El Listado Taxativo

752 actividades del Acuerdo Ministerial 402-2021, embebidas como JSON en la constante `TAX`. Estructura de cada registro y forma de actualizarlo: ver `README.md`.

La cascada es sector → subsector → actividad económica (CIIU) → descripción específica. Al elegir la descripción se muestran el código, el factor de impacto, la unidad de medida y los umbrales de las cinco categorías.

## Deuda conocida

**El catálogo tiene dos defectos, no uno.** Ambos vienen del OCR del PDF oficial del que se derivó la hoja de cálculo.

El primero ya estaba identificado: **los umbrales vienen sucios**. Varias filas traen los rangos cruzados o incompletos — la de remodelaciones dice «C: >500 – 200», que no significa nada. Peor que ilegible es *invertido*: la fila 09-B-013 decía «C: >=500» cuando es «C: <=500». Un rango invertido se lee como válido y categoriza al revés. Por eso **sigue sin haber categorización automática**.

El segundo apareció después y es más difícil de ver: **la actividad económica está arrastrada**. En el PDF oficial esa columna va en celda combinada que cubre varias filas, y el OCR la rellenó hacia abajo con el primer valor. Alrededor de **200 de las 752 filas** quedaron con la actividad de otra. El caso más visible es COMERCIO AL POR MENOR: 19 de sus 20 filas dicen «Venta al por menor de bebidas en comercios especializados», incluidas ferreterías, librerías, armerías y electrodomésticos.

Eso rompía la cascada, porque el paso sector → subsector → **actividad** se apoyaba justo en la columna corrupta. La corrección fue cambiar el camino, no inventar los datos: **ahora se busca por descripción**, que sí quedó bien, y la actividad económica pasó a ser informativa, con un aviso cuando la fila cae en un subsector con arrastre detectado. La detección es automática (`ACT_ARRASTRADA`): si dentro de un subsector una misma actividad cubre el 60% o más de las filas, esa columna no distingue nada.

**Filas verificadas.** La constante `VERIFICADAS` guarda las filas que ya se cotejaron contra el texto del Acuerdo, con su CIIU-4 y sus umbrales correctos, y la interfaz las marca como confiables. Van dos: 09-B-013 (venta al detalle de decorativos, hogar, electrónicos y electrodomésticos — CIIU 4759, C hasta 500 m²) y 09-A-028 (predios de exhibición y venta de vehículos sin taller — CIIU 4510, C hasta 500 m²). Agregar filas ahí conforme se verifiquen es lo que va a desbloquear la categorización automática, y es más barato que limpiar las 752 de una vez: se limpian las que la firma usa.

Los nombres de actividad sí se limpiaron: el OCR generaba variantes corruptas del mismo texto («Construcción de Construc ción tío edificios»), que se agrupan por clave normalizada conservando la variante más limpia.

**Falta limpiar los umbrales contra el PDF original.** Ese es el trabajo que desbloquea la categorización automática.

**Los pasos 5 y 6 son formularios largos.** Ahí es donde una persona sin experiencia se pierde. La descripción del proyecto debería armarse sola a partir de los datos capturados, siguiendo la fórmula de redacción del machote, en vez de pedirle a alguien un párrafo desde cero.

**No genera el formulario en Word ni el anexo fotográfico.** Ese era el entregable original y sigue pendiente.

**No cubre Categoría C con Plan de Gestión Ambiental.** Buena noticia: en SAGA el PGA no es otro formulario, se adjunta como documento al mismo expediente. Se puede agregar como módulo sin rehacer la captura.

## Lo que el MARN exige después

La herramienta nació mirando solo el lado de entrada: lo que el proponente presenta. Las **resoluciones aprobatorias** son el lado de salida, y dicen algo que el instrumento no puede saber solo: qué compromisos impone la autoridad.

De 11 resoluciones del Grupo CODACA/Hino y del lote de Distelsa (2008–2025) salió un patrón estable. Hay un bloque universal que aparece en todas —sea C, C con PGA o B2—, un bloque de fase operativa común a comercio y servicios, uno de fase constructiva para los predictivos, y unos pocos compromisos propios de cada tipología. Están en el paso «Qué genera», en las constantes `COMP_UNIVERSAL`, `COMP_OPERACION`, `COMP_CONSTRUCCION` y `COMP_TIPOLOGIA`.

**Por qué está en la herramienta y no solo en un documento aparte:** un compromiso que la resolución impone y que el instrumento no propuso es un compromiso que el proponente descubre cuando ya lo firmó y no puede negociarlo ni presupuestarlo. Proponerlo desde el instrumento reduce enmiendas y le da al cliente el costo real antes de comprometerse. La regla práctica: si el MARN lo impuso en tres resoluciones de la misma tipología, lo va a imponer en la cuarta.

Ese bloque dice **qué** incluir, nunca **cómo** escribirlo. El registro de una resolución es resolutivo y no es el del consultor; dejarlo entrar a la redacción hace que los instrumentos suenen a MARN.

Dos hallazgos de esas resoluciones que cambian decisiones antes de cotizar:

- **Los umbrales se aplican de verdad.** Una tienda de repuestos de 128 m² se resolvió en B2, veintiocho metros arriba del umbral de 100 m² de la fila 09-A-027. Un taller de 480.79 m² se tramitó como C con PGA, justo debajo de los 500.
- **En una agencia de venta sin taller el MARN impone «no contemplar mantenimiento de vehículos dentro del predio»**, así que conviene declararlo de forma positiva en el instrumento en lugar de callarlo.

## Lo que aprendimos de SAGA

SAGA no es el formulario físico digitalizado. Son **quince pasos**: Datos Generales, Dirección, Información legal, Contacto, Área del proyecto, Fases, Servicios Básicos, cinco pantallas de impactos, Elemento Estético, Requisitos y Previos.

**El paso 15, «Previos», es donde el MARN pide enmiendas**, con fecha de envío y de vigencia — en el expediente revisado, siete días entre una y otra. Los cuatro previos de ese expediente fueron contrato de arrendamiento, patentes, escritura constitutiva y nombramiento del representante legal: **los cuatro legales, ninguno técnico**. Ahí está el retrabajo real, y por eso la herramienta insiste en los documentos legales.

Detalle a tener presente: en la vista de expediente de SAGA **las etiquetas de latitud y longitud están invertidas**. Muestra «Longitud 14°…N / Latitud 90°…W» cuando en Guatemala el valor de 14° es la latitud. Los datos están bien; las etiquetas no.

## Próximos pasos sugeridos

En orden de valor:

1. ~~**La Skill «EAI Categoría C»**~~ — **hecha.** Empaqueta las fórmulas de redacción por tipología (vivienda, comercio, taller y lubricentro, venta de vehículos y repuestos), el motor de coherencia, el catálogo de compromisos ambientales y un generador del formulario oficial en `.docx` con el membrete del MARN. Vive fuera de este repositorio, como skill de Claude.
2. **Autogenerar la descripción del proyecto** a partir de los datos capturados, siguiendo la fórmula de apertura del machote.
3. **Generar el `.docx`** desde esta página. La Skill ya lo hace a partir del mismo `.json` que exporta la herramienta, así que aquí bastaría con enlazarlo o portar el generador.
4. **Seguir verificando filas del taxativo** contra el PDF oficial, agregándolas a `VERIFICADAS`. Cuando estén las que la firma usa a diario, se puede activar categorización automática para esas y solo para esas.
5. **Enlazar desde el clasificador**: cuando el resultado sea Categoría C, ofrecer el enlace a esta herramienta.
6. **Reconstruir la columna de actividad económica** desde el PDF oficial. Es lo único que devolvería la cascada por actividad; mientras tanto la búsqueda por descripción la sustituye bien.
