# Notas de desarrollo

Contexto para retomar el proyecto. Registra por qué las cosas están como están, no solo qué hacen.

Última actualización: agosto 2026 · primera versión publicada.

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

**Los umbrales numéricos vienen sucios.** El catálogo se extrajo de una hoja de cálculo derivada de un OCR del PDF oficial. Varias filas traen los rangos cruzados o incompletos — la de remodelaciones dice «C: >500 – 200», que no significa nada. Por eso **no hay categorización automática**: la herramienta muestra los umbrales como referencia y advierte que deben verificarse contra el texto del Acuerdo. Preferir un veredicto seguro y equivocado sería peor que no dar veredicto.

Los nombres de actividad sí se limpiaron: el OCR generaba variantes corruptas del mismo texto («Construcción de Construc ción tío edificios»), que se agrupan por clave normalizada conservando la variante más limpia.

**Falta limpiar los umbrales contra el PDF original.** Ese es el trabajo que desbloquea la categorización automática.

**Los pasos 5 y 6 son formularios largos.** Ahí es donde una persona sin experiencia se pierde. La descripción del proyecto debería armarse sola a partir de los datos capturados, siguiendo la fórmula de redacción del machote, en vez de pedirle a alguien un párrafo desde cero.

**No genera el formulario en Word ni el anexo fotográfico.** Ese era el entregable original y sigue pendiente.

**No cubre Categoría C con Plan de Gestión Ambiental.** Buena noticia: en SAGA el PGA no es otro formulario, se adjunta como documento al mismo expediente. Se puede agregar como módulo sin rehacer la captura.

## Lo que aprendimos de SAGA

SAGA no es el formulario físico digitalizado. Son **quince pasos**: Datos Generales, Dirección, Información legal, Contacto, Área del proyecto, Fases, Servicios Básicos, cinco pantallas de impactos, Elemento Estético, Requisitos y Previos.

**El paso 15, «Previos», es donde el MARN pide enmiendas**, con fecha de envío y de vigencia — en el expediente revisado, siete días entre una y otra. Los cuatro previos de ese expediente fueron contrato de arrendamiento, patentes, escritura constitutiva y nombramiento del representante legal: **los cuatro legales, ninguno técnico**. Ahí está el retrabajo real, y por eso la herramienta insiste en los documentos legales.

Detalle a tener presente: en la vista de expediente de SAGA **las etiquetas de latitud y longitud están invertidas**. Muestra «Longitud 14°…N / Latitud 90°…W» cuando en Guatemala el valor de 14° es la latitud. Los datos están bien; las etiquetas no.

## Próximos pasos sugeridos

En orden de valor:

1. **La Skill «EAI Categoría C»** — empaquetar la forma de redactar (fórmula de apertura, medidas de mitigación por tipología, enganche a licencia madre) para que funcione en cualquier sesión de Claude, no solo dentro de esta página.
2. **Autogenerar la descripción del proyecto** a partir de los datos capturados.
3. **Generar el `.docx`** con el formato oficial.
4. **Limpiar los umbrales del taxativo** contra el PDF, y recién entonces activar categorización automática.
5. **Enlazar desde el clasificador**: cuando el resultado sea Categoría C, ofrecer el enlace a esta herramienta.
