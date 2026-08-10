# NoA Capture — Captura automática de facturas electrónicas DIAN

Pipeline que lee las facturas electrónicas que llegan al correo de una empresa, extrae los datos
y los escribe estructurados en la hoja contable. Sin digitación manual.

**Estado:** 🟢 en producción en una constructora de Medellín
**Volumen:** 205 facturas procesadas en un mes, corriendo a diario excepto domingos
**Tiempo recuperado:** ~27 horas al mes (328 al año) que antes se iban en digitación
**Cobertura del respaldo:** 56 de esas 205 facturas (27%) llegaron sin XML utilizable y se rescataron leyendo el PDF

> Caso de estudio. El repositorio de cliente es privado; aquí están la arquitectura, las
> decisiones de diseño y los problemas reales que hubo que resolver.

---

## El problema

Las facturas electrónicas llegan al correo de la empresa como archivos `.zip`, decenas al mes,
de proveedores distintos. Alguien las abría una por una, sacaba los datos y los copiaba a dos
hojas de cálculo con formatos diferentes: una de control de facturas y otra en formato contable.

Tres costos: horas de digitación, errores de transcripción, e información que llegaba tarde
para conciliar y cerrar el mes.

## Cómo funciona

```
Correo corporativo (IMAP)
        │
        ▼
   pipeline.py  ←  GitHub Actions, cron diario
        │
        ├─ XML DIAN (UBL 2.1)  ──┐
        │                        ├─►  validación  ─►  deduplicación por CUFE
        └─ PDF vía Claude Vision ┘                             │
                                                               ▼
                                          Hoja FACTURAS  +  Hoja COMPRAS
```

1. Conecta al buzón por IMAP, incluyendo la carpeta de spam (se detecta automáticamente por
   nombre, porque no todos los servidores la llaman igual).
2. Lee los correos no vistos y descarga los adjuntos `.zip`.
3. Extrae el XML DIAN del comprimido y parsea los campos.
4. Descarta los correos que dicen "factura" pero no traen documento adjunto, para no escribir filas vacías.
5. Deduplica contra lo ya escrito, revisando todas las pestañas mensuales.
6. Escribe una fila en la hoja de facturas y otra en la hoja de compras, cada una con su formato.

## Decisiones de diseño

**Se lee el XML, no el PDF.** La factura electrónica DIAN viene con un XML bajo el estándar
UBL 2.1: los campos ya están estructurados y tipados. Hacer OCR sobre el PDF cuando existe el
XML es más frágil, más lento y más caro, sin ninguna ventaja.

**Pero el PDF no es un caso raro: es uno de cada cuatro.** En el primer mes en producción, 56 de
205 facturas llegaron con el ZIP incompleto o con campos críticos vacíos en el XML. Cuando eso
pasa, el pipeline cae al PDF y lo lee con Claude Sonnet en modo visión. Sin ese respaldo el
sistema resolvería tres cuartas partes del problema y las otras 56 facturas seguirían
digitándose a mano cada mes.

**Nunca se marca un correo como leído.** El equipo de la empresa usa el mismo buzón. Si el
proceso automático marcara los correos, la gente perdería la referencia de qué ha revisado.
La lectura se hace con `BODY.PEEK[]` en lugar de `BODY[]`, que trae el contenido sin activar
la bandera `\Seen`. Fue un requisito no negociable del cliente y condicionó todo el acceso IMAP.

**Se revisa también la carpeta de spam.** Una parte de las facturas electrónicas termina ahí por
el filtro del proveedor de correo, y nadie revisa el spam a mano. Esas facturas no se digitan
tarde: se pierden. El pipeline detecta la carpeta automáticamente por nombre, porque no todos
los servidores la llaman igual.

**Idempotencia por CUFE.** El campo tiene 69 caracteres, así que en el proceso manual nadie lo
transcribía — la deduplicación no era lenta, era imposible. Deduplicar por CUFE permite reprocesar un lote completo sin miedo a duplicar filas — y los
proveedores reenvían facturas más seguido de lo que uno esperaría.

**Despliegue en GitHub Actions, no en un servidor.** Es un trabajo por lotes que corre una vez
al día. Levantar y pagar un servidor para eso es sobre-ingeniería: un cron en Actions cuesta
cero y ya trae logs y reintentos.

## Stack

| Componente | Tecnología | Por qué |
|---|---|---|
| Lenguaje | Python | Parsing XML y manejo de correo |
| Correo | IMAP (`BODY.PEEK[]`) | Lectura no destructiva del buzón |
| Datos fuente | XML DIAN UBL 2.1 | Campos ya estructurados |
| Respaldo | Claude Sonnet (visión) | Rescata las facturas con XML incompleto |
| Destino | Google Sheets | El contador ya trabajaba ahí; cero fricción de adopción |
| Ejecución | GitHub Actions (cron) | Sin infraestructura que mantener |

## Qué aprendí

**La norma dice una cosa y el correo trae otra.** La factura electrónica DIAN se entrega, por
norma, en un ZIP con el XML y el PDF, los dos con la misma información. Diseñé el pipeline
leyendo únicamente el XML, porque viene estructurado y tipado y es la fuente correcta. En
producción aparecieron facturas que llegaban solo con PDF, y esas sencillamente no se estaban
leyendo. Agregué la extracción desde el PDF con un modelo de visión y en el primer mes eso
rescató 56 de 205 facturas. Cumplir el estándar es responsabilidad de quien emite; no se puede
construir asumiendo que todos lo cumplen.

**Filtrar por el asunto no alcanza.** La condición de entrada terminó siendo doble: que el
asunto contenga la palabra "factura" y que el correo traiga adjunto. El asunto declara la
intención, el adjunto confirma que hay algo que leer. Si dice factura y no trae nada, no se
procesa — porque una fila vacía en la entrega al contador es peor que no tener fila.

---

Construido por [Juan Fernando Toro Isaza](https://github.com/juanftoro2006) — automatización de procesos para pymes.
