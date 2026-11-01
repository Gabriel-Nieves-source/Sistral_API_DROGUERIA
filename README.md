# SITRAL API — Sistema de Trazabilidad y Control por Lotes para Droguerías

Backend REST en **Python + FastAPI** para la app móvil de SITRAL. Registra lotes de medicamentos por escaneo
(EAN, GS1 y QR), genera **alertas de vencimiento** con una ventana crítica estricta de **10 días**, integra
**datos abiertos colombianos** (INVIMA y DANE) y ofrece **analítica con DuckDB** en un dashboard web.

> Guía de Laboratorio 6 (Sesiones 11 y 12) — Taller de Programación, Ingeniería de Software, Universidad Manuela Beltrán.

| | |
|---|---|
| **Integrantes** | Cristian Camilo Girata Ravelo · Gabriel Leonardo Nieves Bello · Zharick Yojana Puentes Rodríguez |
| **Tutor** | Juan José Osorio Tabares |
| **Video (inglés)** | _[enlace de YouTube]_ |

## Arquitectura hexagonal

```
app/
├── domain/          # Reglas puras y puertos (typing.Protocol), sin frameworks
│   ├── inventario/  # Lotes, regla de 10 días, escaneo GS1/QR
│   ├── alertas/     # Alertas PREVENTIVA, CRITICA y VENCIDO
│   ├── asistente/   # Puertos del LLM (RAG)
│   └── datos_abiertos/
├── application/     # Casos de uso con inyección de dependencias por constructor
├── infrastructure/  # Adaptadores: SQLAlchemy, Gemini, datos.gov.co, circuit breaker, DuckDB
└── api/             # FastAPI: routers, esquemas, dashboard y raíz de composición (deps.py)
```

- **Las dependencias apuntan hacia el dominio:** el núcleo no importa FastAPI, SQLAlchemy, httpx ni DuckDB.
- **Inyección por constructor:** por ejemplo, `ServicioAlertas(lotes, alertas, configuracion, uow)`.
- **Bounded contexts (DDD):** Inventario, Alertas, Asistente IA y Datos abiertos.
- **Decisiones de arquitectura:** cuatro ADR en inglés en [`docs/adr/`](docs/adr/README.md).

## Datos abiertos colombianos y resiliencia

| Fuente | Dataset | Uso |
|---|---|---|
| INVIMA — Código Único de Medicamentos (CUM) | `i7cb-raxc` | Verificar registros sanitarios y medir la oferta por principio activo |
| DANE — DIVIPOLA | `gdxc-w37w` | Ubicar proveedores por municipio |

Cada llamada pasa por un **circuit breaker** de 3 estados, **reintentos con backoff exponencial** y una **caché JSON**:

`CLOSED` → 3 fallos seguidos → `OPEN` (responde desde la caché) → 20 s → `HALF_OPEN` → la llamada de prueba funciona → `CLOSED`

## Analítica con DuckDB

| Endpoint | Análisis |
|---|---|
| `/api/v1/analitica/inventario` | Lotes, unidades y valor por nivel de vencimiento: media, mediana, desviación estándar, P25 y P75 |
| `/api/v1/analitica/oferta-mercado` | Presentaciones y titulares registrados en el INVIMA por principio activo |
| `/api/v1/analitica/cobertura-regional` | Lotes en riesgo por municipio del proveedor (cruce con DIVIPOLA) |

Los resultados se ven en **http://127.0.0.1:8000/dashboard**.

## Instalación y ejecución (Windows)

```powershell
cd sitral-api
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
```

En `.env` configure como mínimo:
```
DATABASE_URL=sqlite:///./sitral.db
JWT_SECRET_KEY=<clave generada con: python -c "import secrets; print(secrets.token_urlsafe(48))">
GEMINI_API_KEY=<opcional, solo para el asistente IA>
```

Luego:
```powershell
python -m scripts.crear_admin --usuario admin --nombre Ana --apellido Rojas
python -m scripts.sembrar_datos_demo        # datos ficticios de demostración
python -m scripts.explorar_datos_abiertos   # verifica la conexión con datos.gov.co
python main.py
```

| URL | Contenido |
|---|---|
| http://127.0.0.1:8000/docs | Swagger UI (documentación en inglés) |
| http://127.0.0.1:8000/dashboard | Dashboard analítico y estados del circuit breaker |

## Pruebas

```powershell
pytest -v
```

**104 pruebas aprobadas.** Incluyen pruebas tipo **Mockito** con `unittest.mock` sobre los casos de uso, los tres
estados del circuit breaker con un reloj simulado, y los adaptadores con un servidor HTTP simulado. Ninguna prueba
llama a internet ni al LLM. GitHub Actions ejecuta la suite en cada *push*.

## Seguridad

- Contraseñas con **Argon2id** y autenticación **JWT** con roles `ADMINISTRADOR` y `FARMACEUTA`.
- **Nunca suba al repositorio** el archivo `.env`, la carpeta `.venv` ni la base `sitral.db`; ya están en `.gitignore`.

## Uso de IA

El código se desarrolló con apoyo de **Claude (Anthropic)** y **GitHub Copilot**. El equipo revisó, ejecutó y
validó todo el código. La evaluación de una sugerencia de Copilot está documentada en el informe de laboratorio.
