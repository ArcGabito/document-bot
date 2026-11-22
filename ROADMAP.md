# Plan por fases: DocumentBot

> **Objetivo**: que DocumentBot automatice la **gestión documental** de la empresa. Captura los documentos desde cualquier canal, los clasifica y extrae sus datos con OCR e IA, los valida, los archiva en expedientes con permisos y retención, pide los que faltan, avisa los vencimientos y genera y firma documentos desde plantillas. Debe desplegarse con la misma imagen en un **VPS** (Docker Compose) o en **Azure Container Apps**.
>
> Stack: **Python 3.13 + uv · FastAPI · SQL Server / Azure SQL · Redis · colas (Dramatiq) · Docker**.

## Punto de partida

**Existe**, extraído de invoice-bot:

- Lectura de PDF (PDFium) y descubrimiento de archivos con SHA-256 por streaming.
- Base técnica: conexión a SQL Server (cadena de SSMS, seguridad integrada, identidad administrada), runner de migraciones con copia previa, candado, usuarios, seguridad de la API, inicio de sesión, logs JSON y `Caddyfile`.

**Falta**:

- `pyproject.toml`, `uv.lock`, `config.py`, `cli.py`, la aplicación FastAPI y las capas `domain` y `application`.
- OCR, esquema de base de datos, pruebas, Dockerfile, Compose y CI.
- Módulos que el código ya importa pero no existen: `documentbot.config` y `documentbot.infrastructure.db.rows`.
- El primer push: el repositorio de GitHub está vacío.

**Alcance**: invoice-bot ya cubre los comprobantes de pago (facturas, boletas y notas). DocumentBot cubre **todo lo demás**: contratos, órdenes de compra, guías de remisión, recibos por honorarios, DNI, fichas RUC, vigencias de poder, pólizas, cartas fianza, certificados, CV y documentos laborales. Si recibe un comprobante, lo deriva a invoice-bot.

## Arquitectura objetivo

```text
 escáner / carpeta / correo (email-bot) / API / móvil
                     │
          ┌──────────▼──────────┐   encola    ┌──────────────── Redis ────────────────┐
 Caddy ─► │ api (FastAPI) x N   │ ──────────► │ colas · caché · candados · rate limit │
 HTTPS    │ sin estado          │             └───┬───────────────────────▲───────────┘
          └──────────┬──────────┘                 │ consume               │ encola (cron)
                     │                            ▼                       │
                     │                 ┌────────────────────┐   ┌─────────┴──────────┐
                     └───────────────► │ SQL Server + blob  │◄──│ workers: ocr,      │
                                       │ (metadatos, texto, │   │ extract, render    │
                                       │  archivos por hash)│   │ scheduler (1 líder)│
                                       └────────────────────┘   └─────────┬──────────┘
                                                                          ▼
                                                               Gotenberg (DOCX → PDF)
```

| Proceso | Comando | VPS (Docker Compose) | Azure Container Apps |
| --- | --- | --- | --- |
| `api` | `documentbot serve` | servicio `api` tras Caddy | Container App con ingress, escala por HTTP (mín. 1) |
| `worker` | `documentbot worker --queues ingest,extract,default` | servicio `worker`, `--scale worker=N` | Container App sin ingress, de 0 a N con KEDA (`redis`) |
| `worker-ocr` | `documentbot worker --queues ocr` | servicio aparte con más CPU y RAM (OCRmyPDF y Tesseract) | Container App con perfil de más CPU, de 0 a N con KEDA |
| `worker-render` | `documentbot worker --queues render` | servicio aparte | Container App de 0 a N con KEDA |
| `gotenberg` | — | contenedor `gotenberg/gotenberg` interno (LibreOffice sin interfaz) | Container App interno |
| `scheduler` | `documentbot scheduler` | 1 réplica, candado de líder en Redis | Container Apps Job programado: `documentbot schedule tick` cada 5 min |
| `migrate` | `documentbot db migrate` | `docker compose run --rm api db migrate` | Container Apps Job manual |
| Redis | — | `redis` con AOF y `noeviction` | Azure Managed Redis |
| SQL Server | — | perfil `sql` (Express) o servidor externo | Azure SQL serverless con identidad administrada |
| Archivos | — | volumen o MinIO (S3) | Azure Blob Storage con identidad administrada |

### Decisiones técnicas

| Tema | Decisión | Motivo |
| --- | --- | --- |
| Dependencias | **uv** con `uv.lock`, `.python-version` y `[dependency-groups] dev` | Compilaciones reproducibles y rápidas |
| Colas | **Dramatiq + Redis**, con colas separadas por tipo de carga (`ocr`, `extract`, `render`) | El OCR consume mucha CPU y no debe frenar la ingesta ni la generación. Cada pool escala por separado con KEDA |
| Almacenamiento | Binarios en blob direccionados por SHA-256; metadatos, texto OCR y datos extraídos en SQL Server | Sin duplicar archivos. La búsqueda de texto completo queda en la base |
| Datos por tipo | **JSON Schema por tipo de documento**, versionado y editable desde la pantalla | Agregar un tipo nuevo no requiere programar |
| Extracción | Primero lo determinista (XML, QR, código de barras, PDF con texto) y después el LLM con salida estructurada | Más barato y más exacto, con la misma filosofía que invoice-bot: la IA propone y el código valida |
| Consistencia | **Outbox transaccional** para eventos | Sin pérdidas si Redis cae |
| Modo sin Redis | `DOCUMENTBOT_QUEUE_BACKEND=inline` | Pruebas y desarrollo |

### Colas de DocumentBot

| Cola | Actores | Límites |
| --- | --- | --- |
| `ingest` | `files.discover(source_id)`, `document.store(upload_id)`, `batch.split(document_id)` | Un descubrimiento por carpeta a la vez |
| `ocr` | `document.ocr(document_id)`: rotación, enderezado, OCR y PDF/A buscable | Concurrencia según CPU; tiempo límite por página |
| `extract` | `document.classify(document_id)`, `document.extract(document_id)`, `document.validate(document_id)` | Rate limit de tokens de la IA |
| `render` | `template.render(job_id)`, `batch.render(batch_id)`, `document.sign(document_id)` | Concurrencia limitada por Gotenberg |
| `default` | `expirations.check()`, `requests.remind()`, `retention.apply()`, `webhooks.deliver(event_id)`, `outbox.relay()`, `db.backup()` | — |

---

## Fase 0: Base del proyecto y plataforma

**Objetivo**: un proyecto que arranca, se prueba, se publica y se despliega, con la misma base que invoice-bot.

- [ ] `pyproject.toml` con uv (`uv init --package`), `uv.lock`, `.python-version` (3.13) y grupo `dev` (pytest, ruff, mypy).
- [ ] Crear los módulos faltantes: `config.py` (variables `DOCUMENTBOT_*`), `context.py`, `cli.py` (Typer), `api/app.py` e `infrastructure/db/rows.py`.
- [ ] **Esquema inicial** (migración v1): `documents`, `document_pages`, `document_types`, `extractions`, `files` (por SHA-256), `users`, `review_sessions`, `api_keys`, `jobs`, `outbox_messages` y `bot_runs`.
- [ ] **Almacenamiento** con una interfaz única y tres implementaciones: disco local, S3/MinIO y Azure Blob.
- [ ] **Redis + Dramatiq**: broker, actores `ingest` y `default`, y comandos `documentbot worker` y `documentbot scheduler`.
- [ ] **Dockerfile** multietapa con uv, ODBC 18, Tesseract (español e inglés), OCRmyPDF y Ghostscript, con usuario sin privilegios y `HEALTHCHECK`. `docker-compose.yml` con `api`, `worker`, `worker-ocr`, `worker-render`, `gotenberg`, `scheduler`, `redis` y los perfiles `sql`, `minio` y `proxy`. Bicep para Container Apps (API, workers con KEDA, Jobs, Managed Redis, Blob y Key Vault).
- [ ] `/healthz`, `/readyz` (SQL Server, esquema, Redis y almacenamiento), `/metrics` y `documentbot doctor` (OCR, Gotenberg, IA y almacenamiento).
- [ ] **CI/CD**: `astral-sh/setup-uv`, ruff, mypy, pytest con SQL Server, Redis y MinIO como servicios, imagen en GHCR, Trivy y despliegue con OIDC. Primer push con README, `.env.example` y `docs/`.

**Hecho cuando**: subir un PDF por la API lo guarda una sola vez (por SHA-256) y lo encola, y la imagen corre en un VPS y en Container Apps.

## Fase 1: Captura y OCR

**Objetivo**: que cualquier documento, venga de donde venga, quede como un PDF/A buscable con su texto en la base.

- [ ] **Canales**:
  - carpeta vigilada (escáner de red, OneDrive o Azure Files);
  - API (multipart y URL prefirmada para archivos grandes);
  - email-bot (adjuntos clasificados como documento);
  - foto desde el móvil (corrección de perspectiva).
- [ ] **Preprocesado**: rotación, enderezado, limpieza y detección de páginas en blanco.
- [ ] **OCR** con Tesseract mediante OCRmyPDF, que genera un **PDF/A buscable** y conserva el original. Opcionalmente, visión de Claude para fotos difíciles o manuscritos.
- [ ] **Separación de lotes**: un escaneo de 200 hojas se divide en documentos por hoja separadora, QR o código de barras, o por cambio de documento detectado por la IA.
- [ ] Texto por página con coordenadas (*bounding boxes*) para resaltar en la revisión de dónde salió cada dato.

**Hecho cuando**: un lote escaneado de varios documentos se separa, se procesa con OCR y queda buscable por texto, con su original intacto.

## Fase 2: Clasificación, extracción y validación

**Objetivo**: saber qué es cada documento y tener sus datos listos sin teclear.

- [ ] **Catálogo inicial de tipos**: contrato, adenda, orden de compra, guía de remisión, recibo por honorarios, DNI y carné de extranjería, ficha RUC, vigencia de poder, póliza de seguro, carta fianza, certificado (ITSE, calibración, capacitación), CV, boleta de pago y declaración jurada. Cada tipo trae su JSON Schema.
- [ ] **Extracción determinista primero**:
  - la **guía de remisión electrónica** se lee de su XML UBL (DespatchAdvice), sin IA;
  - los QR y códigos de barras se decodifican directamente;
  - los PDF con texto se leen sin OCR.
- [ ] **LLM** con salida estructurada validada contra el esquema del tipo, más la confianza por campo.
- [ ] **Validaciones deterministas**: RUC (módulo 11) y estado en SUNAT, DNI de 8 dígitos, fechas coherentes (inicio antes del fin, documento no vencido), montos y monedas, y campos obligatorios por tipo.
- [ ] **Bandeja de revisión** con el documento y los datos lado a lado, el resaltado del origen y la corrección con auditoría. Los comprobantes de pago se derivan a invoice-bot.

**Hecho cuando**: sobre un conjunto etiquetado de documentos reales, la precisión por tipo y por campo se mide en CI y no baja entre versiones.

## Fase 3: Gestión documental (expedientes)

**Objetivo**: reemplazar las carpetas compartidas por expedientes ordenados, buscables y con permisos.

- [ ] **Expedientes** por cliente, proveedor, trabajador, contrato o proyecto, con asignación automática según los datos extraídos (RUC, DNI, número de contrato).
- [ ] Metadatos, etiquetas, **versiones** (un documento nuevo reemplaza al anterior y lo conserva) y relaciones (contrato ↔ adendas ↔ cartas fianza).
- [ ] **Búsqueda** de texto completo (SQL Server Full-Text Search sobre el OCR) con filtros por tipo, fecha, expediente y estado.
- [ ] **Permisos** por empresa, área y expediente (RR. HH. ve legajos, Logística ve guías), SSO OIDC (Microsoft Entra ID y Google Workspace) y Row-Level Security.
- [ ] **Retención y eliminación** por tipo (por ejemplo, los legajos laborales durante el plazo legal), con retención legal que bloquea el borrado y registro de cada eliminación.
- [ ] **Auditoría de accesos**: quién vio, descargó o compartió cada documento.
- [ ] Enlaces para compartir con caducidad y **redacción**: copias con datos personales tachados.

**Hecho cuando**: el legajo de un trabajador o el expediente de un proveedor se arma solo con los documentos que llegan, y cada acceso queda registrado.

## Fase 4: Automatización de procesos documentales

**Objetivo**: que los procesos que dependen de papeles avancen solos.

- [ ] **Onboarding de proveedores, clientes y trabajadores**:
  - lista de documentos requeridos por tipo de expediente (por ejemplo, para un proveedor: ficha RUC, vigencia de poder, cuenta bancaria, homologación);
  - pedido automático de los faltantes por correo (email-bot) con un enlace de carga, y recordatorios;
  - el expediente queda "completo" solo cuando todo está validado.
- [ ] **Control de vencimientos** de contratos, pólizas, cartas fianza, certificados ITSE, licencias, SOAT, certificados médicos y homologaciones. Alertas escalonadas (60, 30 y 7 días) al responsable y a su jefe.
- [ ] **Contratos**: extracción de cláusulas clave (plazo, renovación automática, preaviso, penalidades, garantías) y alerta **antes del plazo de preaviso**, no del vencimiento.
- [ ] **Recepción de mercadería**: las guías de remisión validadas se envían a invoice-bot para el match a 3 vías (orden de compra ↔ guía ↔ factura).
- [ ] **Vigencias en línea**: reconsulta programada en SUNAT del estado de los RUC de los expedientes activos.

**Hecho cuando**: un proveedor nuevo completa su expediente sin que nadie le escriba a mano, y ningún contrato se renueva sin aviso.

## Fase 5: Generación y firma de documentos

**Objetivo**: emitir los documentos de la empresa desde datos, no desde Word.

- [ ] **Plantillas DOCX** con variables (docxtpl), versionadas y con vista previa, para contratos, cartas, constancias de trabajo, certificados, actas, cotizaciones y cartas de cobranza.
- [ ] **Conversión a PDF** con Gotenberg, en un contenedor aparte.
- [ ] **Generación masiva** desde la API, Excel o el ERP (por ejemplo, 500 constancias), con numeración correlativa por serie.
- [ ] **QR de verificación**: página pública que confirma, con el hash, que el documento es auténtico y no fue modificado.
- [ ] **Firma digital PAdES** con certificado digital (pyHanko), sello de tiempo (TSA) y verificación de firmas de documentos recibidos. Integración opcional con proveedores de firma electrónica para firmas de terceros.
- [ ] **Flujo de firma**: orden de firmantes, recordatorios y el documento firmado archivado en el expediente. El envío sale por email-bot.

**Hecho cuando**: un contrato se genera desde los datos del expediente, se firma y se archiva con su QR de verificación sin salir de DocumentBot.

## Fase 6: Integración y analítica

- [ ] **Webhooks firmados** (HMAC-SHA256) en formato CloudEvents: `document.received`, `document.classified`, `document.validated`, `document.expiring`, `document.missing` y `document.signed`.
- [ ] **API de consulta** para el ERP y el CRM (documentos de un cliente o proveedor y estado de su expediente).
- [ ] **Indicadores** en vistas SQL para Power BI y en un panel propio: documentos por tipo y canal, porcentaje de extracción sin revisión, tiempo de revisión, expedientes incompletos, documentos por vencer y costo de IA por documento.

## Fase 7: Producción a escala

- [ ] **Observabilidad** con OpenTelemetry (FastAPI, Dramatiq, httpx y SQL) hacia Azure Monitor o Grafana. Alertas por cola de OCR creciendo, DLQ, Gotenberg caído y documentos estancados.
- [ ] **Alta disponibilidad y recuperación**: API con 2 o más réplicas, Redis con persistencia y réplica, Azure SQL con redundancia de zona o copias probadas, blob con versionado y *soft delete* (o réplica de MinIO) y un runbook con RPO y RTO.
- [ ] **Despliegues sin cortes** (revisiones de Container Apps o actualización gradual en el VPS) y migraciones como job previo.
- [ ] **Seguridad**: Renovate o Dependabot para `uv.lock`, Trivy, SBOM, OWASP ASVS nivel 2, análisis seguro de archivos (límites de páginas y tamaño, PDF con JavaScript bloqueado, imágenes "bomba") y secretos solo en Key Vault.
- [ ] **Pruebas de carga**: 20 000 páginas OCR por día como referencia, para dimensionar `worker-ocr`.

---

## Integración con la suite de bots

Cada bot tiene **su propia base de datos** y se despliega por separado. Se comunican por **API REST con claves** y **webhooks firmados** (HMAC-SHA256) en formato CloudEvents, publicados desde la outbox.

| Evento | Emisor → receptor | Efecto |
| --- | --- | --- |
| `email.attachment.received` (documento) | email-bot → **document-bot** | Clasifica y archiva |
| `document.classified` (comprobante de pago) | **document-bot** → invoice-bot | Lo procesa invoice-bot |
| `document.classified` (guía de remisión) | **document-bot** → invoice-bot | Recepción para el match a 3 vías |
| `document.expiring` / `document.missing` | **document-bot** → email-bot | Alerta de vencimiento o pedido de documentos |
| `document.signed` | **document-bot** → email-bot | Envío del documento firmado |
