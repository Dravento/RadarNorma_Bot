# RadarNorma Bot 🤖📋

**Asistente inteligente de monitoreo normativo para estudios contables**

> Trabajo Final Integrador · Diplomatura en Diseño de Procesos de Negocios con Plataformas Visuales de Automatización e IA Agéntica · FCE-UBA 2026  
> Autora: Aylen Delgado

---

## ¿Qué hace?

RadarNorma Bot monitorea automáticamente la Primera Sección del Boletín Oficial de la República Argentina todos los días hábiles, analiza cada norma publicada con inteligencia artificial, la cruza contra la cartera de clientes del estudio contable y envía una notificación personalizada por Telegram **solo cuando hay algo que el contador realmente necesita saber**.

El problema que resuelve es concreto: el Boletín Oficial publica entre 80 y 150 normas por día hábil. Un contador puede dedicar entre 30 y 60 minutos diarios solo a leer, filtrar y cruzar esa información con sus clientes — de forma manual, dependiendo de su memoria y disponibilidad. RadarNorma automatiza ese proceso de punta a punta.

---

## ¿Cómo funciona?

El sistema tiene dos workflows independientes construidos en n8n:

### Workflow 1 — Monitoreo diario del Boletín Oficial

Se ejecuta automáticamente de lunes a viernes a las 9:00 AM (hora Argentina).

```
Schedule Trigger (L-V 9AM ARG)
  → Scraping del BO con paginación automática
  → Filtro por organismos relevantes (ARCA, BCRA, Ministerio de Economía, ANSES, etc.)
  → Extracción de texto limpio de cada norma
  → Análisis con GPT-4o-mini → JSON estructurado
  → Matching norma × cartera de clientes (scoring multidimensional)
  → Notificación agrupada por Telegram
```

Cada notificación incluye: título, organismo, resumen ejecutivo, nivel de importancia, **vigencia** (inmediata / diferida / pendiente de reglamentación), acción sugerida, clientes afectados y link directo a la norma.

### Workflow 2 — Onboarding de clientes desde Excel

Permite cargar nuevos clientes con solo enviar un Excel con CUITs al bot.

```
Telegram (archivo Excel con CUITs)
  → Descarga y lectura del Excel
  → Consulta automática a ARCA via ARCANUM (razón social, actividad, impuestos, provincia, cierre)
  → UPSERT en Supabase
  → Confirmación por Telegram
```

---

## Stack tecnológico

| Herramienta | Rol |
|---|---|
| **n8n** (Docker) | Orquestador principal de ambos workflows |
| **GPT-4o-mini** (OpenAI) | Análisis y clasificación de normas fiscales |
| **Telegram Bot API** | Canal de notificaciones (@RadarNorma_Bot) |
| **Supabase** | Base de datos de clientes y certificados |
| **ARCANUM** | Gateway open source para consultar el padrón de ARCA |
| **ngrok** | Exposición pública del webhook de Telegram (desarrollo) |
| **Boletín Oficial API** | Fuente de datos con paginación automática |

---

## Algoritmo de matching

Cada norma se cruza contra todos los clientes de la base usando un sistema de scoring:

| Dimensión | Puntos |
|---|---|
| Impuestos en común | +3 |
| Tipo de contribuyente coincide | +2 |
| Actividad económica específica | +2 |
| Provincia (si la norma es provincial) | +1 |
| Provincia no coincide | descarte |
| Tipo contribuyente no coincide | descarte |
| Cierre de ejercicio no coincide | descarte |

**Umbral mínimo para notificar: 4 puntos.**

Además, antes del matching se aplican tres filtros: normas con importancia "nula" se descartan, normas dirigidas a un contribuyente específico se descartan, y normas sin impuestos afectados y con importancia baja/media se descartan.

---

## Estructura del repositorio

```
RadarNorma_Bot/
├── README.md
├── workflows/
│   ├── radarnorma_monitoreo_diario.json     # Workflow 1 — exportado de n8n
│   └── radarnorma_onboarding_clientes.json  # Workflow 2 — exportado de n8n
└── plantillas/
    └── clientes_radarnorma.xlsx             # Plantilla Excel para carga de clientes
```

---

## Cómo usar

### Requisitos previos

- Docker Desktop instalado
- Cuenta de OpenAI con acceso a GPT-4o-mini
- Bot de Telegram creado via @BotFather
- Proyecto en Supabase con las tablas `Clientes RadarNorma` y `certificados`
- ARCANUM corriendo en Docker (puerto 8094)
- ngrok para exponer el webhook localmente

### Tabla de clientes en Supabase

```sql
CREATE TABLE "Clientes RadarNorma" (
  id bigint generated always as identity primary key,
  cuit text unique not null,
  razon_social text,
  actividad text,
  provincia text,
  impuestos text,
  tipo text,
  cierre int4
);
```

### Importar los workflows

1. Abrí n8n en `http://localhost:5678`
2. Menú → Import workflow
3. Importá los dos archivos JSON de la carpeta `workflows/`
4. Configurá las credenciales: OpenAI, Telegram, Supabase y HTTP Request para ARCANUM

### Plantilla de clientes

El archivo `plantillas/clientes_radarnorma.xlsx` contiene el formato esperado para la carga de clientes. Solo es necesario completar la columna **CUIT** — el sistema obtiene el resto desde ARCA automáticamente.

---

## Seguridad

- Los datos de clientes se almacenan en Supabase con Row Level Security habilitado.
- El certificado digital de ARCA y la clave privada se guardan cifrados con **AES-256-CBC** en Supabase.
- La master key de cifrado vive únicamente como variable de entorno en Docker (`RADARNORMA_MASTER_KEY`) y nunca se exporta al código ni al repositorio.
- El texto de las normas (información pública del BO) es el único dato enviado a OpenAI. Los datos de clientes nunca salen del servidor local.

---

## Limitaciones conocidas (v1)

- La URL de ngrok cambia al reiniciar, requiriendo reconfigurar el webhook de Telegram. En producción se resuelve con un servidor con IP fija.
- Para empresas grandes (SA, bancos), ARCANUM puede devolver el campo de actividad vacío debido a una diferencia en la estructura del JSON de ARCA. Pendiente para v2.
- Si se envía el mismo Excel dos veces, el UPSERT evita duplicados en la base pero puede generar un mensaje de confirmación duplicado en Telegram.

---

## Roadmap

| Versión | Mejoras planificadas |
|---|---|
| v1 (actual) | Monitoreo BO Nacional, matching por cliente, onboarding via ARCA, notificación Telegram |
| v2 | Botones interactivos en Telegram, email formal al cliente, resumen semanal, fix actividad SA |
| v3 | Normativa ARCA directa, Comisión Arbitral (IIBB interjurisdiccional), organismos provinciales |
| v4 | Vencimientos fiscales personalizados, dashboard web |
| SaaS | Servidor en nube, multi-tenant, modelo de suscripción |

---

## Contexto académico

Este proyecto fue desarrollado como Trabajo Final Integrador de la **Diplomatura en Diseño de Procesos de Negocios con Plataformas Visuales de Automatización e IA Agéntica** — FCE-UBA, Cohorte 2026, bajo la instrucción del Prof. Diego Parrás.

El proceso automatizado reemplaza una tarea manual de 30-60 minutos diarios por un ciclo automatizado de 3-5 minutos, con notificaciones personalizadas por cliente y trazabilidad completa de cada norma analizada.
