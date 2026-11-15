# Asistente de verificación de datos (fact-checking) para redacciones

Propuesta de TFM para un máster de desarrollo con IA. Sistema que ayuda a periodistas a verificar afirmaciones cuantitativas: extrae las afirmaciones comprobables de un texto, busca evidencias en fuentes fiables y propone un veredicto razonado y citado. **El veredicto final lo decide siempre el periodista.**

**Stack:** Python (FastAPI) · Vue.js · Google ADK · MCP · RAG (OpenSearch) · PostgreSQL · Docker Compose · TDD

---

## 1. Objetivo y alcance

- **Problema:** verificar una afirmación consume mucho tiempo (localizar la fuente primaria, comparar cifras, comprobar si ya se verificó, documentar el proceso).
- **Solución:** un sistema que recibe un texto y devuelve fichas de verificación por afirmación, con evidencias citadas, veredicto propuesto y nivel de confianza.
- **Alcance recomendado:** afirmaciones **cuantitativas y comprobables con datos abiertos** (por ejemplo, «el paro juvenil bajó un 10 % en 2024»). Son las más fáciles de evaluar de forma objetiva y evitan zonas grises de opinión.

## 2. Flujo de uso

1. El periodista pega un texto o una URL.
2. El sistema extrae y resalta las afirmaciones verificables.
3. Para cada una, el agente busca evidencias (datos oficiales, verificaciones previas, hemeroteca).
4. Se genera una ficha con veredicto, confianza, evidencias a favor y en contra, y lo que no se pudo comprobar.
5. El periodista revisa, edita, acepta o descarta, y la decisión queda registrada.

## 3. Arquitectura

```
Vue.js (Pinia + SSE)
      │
FastAPI ── ADK Runner ── Agente(s) ADK ── (cliente MCP) ──► Servidor MCP
      │                                                       ├─ datos oficiales (INE, Eurostat…)
      │                                                       ├─ búsqueda RAG (OpenSearch)
      │                                                       └─ registro de veredictos
      ▼
PostgreSQL (fichas, decisiones, auditoría)
```

Todo se despliega con `docker compose`: `frontend`, `api`, `mcp-server`, `opensearch` y `postgres`.

## 4. RAG

**Corpus (fuentes abiertas):**
- Verificaciones previas y artículos de referencia de uso legítimo.
- Notas metodológicas de estadísticas oficiales (qué mide cada indicador y cómo se define), útiles para detectar afirmaciones engañosas por definición.
- Hemeroteca simulada, si se quiere imitar el archivo de un periódico.

**Recuperación:**
- Troceado por unidades con sentido (párrafo o sección), conservando metadatos: fuente, fecha, tipo y fiabilidad.
- **Búsqueda híbrida** (BM25 + vectorial), porque en verificación importan tanto los números, nombres y fechas exactos como la semántica.
- Filtros por fecha, para comparar cada afirmación con datos de su mismo periodo.
- Cada fragmento lleva un identificador para poder citarlo con exactitud.

## 5. Servidor MCP

| Herramienta | Función |
|---|---|
| `buscar_evidencias(consulta, filtros)` | Búsqueda RAG; devuelve fragmentos con fuente y fecha |
| `consultar_dato_oficial(indicador, periodo, territorio)` | Obtiene una serie desde una API de datos abiertos (INE, Eurostat) |
| `comparar_cifras(declarada, oficial, tolerancia, unidad)` | Comparación determinista con normalización de unidades y redondeos |
| `buscar_verificaciones_previas(afirmacion)` | Detecta si el caso ya se verificó |
| `registrar_veredicto(ficha)` | Persiste la ficha y la decisión humana |

**Principio de diseño:** todo lo que pueda hacerse con código determinista (comparar cifras, normalizar fechas) va en el MCP y no en el LLM. Reduce errores y facilita el TDD.

## 6. Agentes con Google ADK

Pipeline orquestado con un `SequentialAgent` y estado compartido de sesión:

- **`ExtractorAgent`:** devuelve la lista estructurada de afirmaciones (esquema Pydantic): texto, tipo, entidades y periodo.
- **`InvestigadorAgent`:** uno por afirmación, en paralelo (`ParallelAgent`); planifica consultas y usa las herramientas MCP.
- **`JuezAgent`:** propone veredicto, confianza y justificación, citando identificadores de evidencia concretos.
- **`RedactorAgent`:** compone la ficha final legible.

**Puntos de diseño con valor académico:**
- **Abstención:** poder responder «no verificable con las fuentes disponibles».
- **Escala de veredictos documentada:** *Cierto, Mayormente cierto, Engañoso, Falso, No verificable*.
- **Callbacks de ADK** que validan que cada cita corresponde a una evidencia realmente recuperada (anti-alucinación de citas).

## 7. Frontend (Vue.js)

- Texto con afirmaciones resaltadas por color según el veredicto.
- Panel lateral con la ficha: evidencias a favor y en contra, fuentes enlazadas, confianza y razonamiento.
- Progreso en tiempo real (SSE) mientras trabaja el agente.
- Aceptar, corregir o rechazar la propuesta; la decisión se guarda.
- Historial de verificaciones y exportación de la ficha.
- Pinia para el estado, Vitest para tests unitarios y Playwright para pruebas de extremo a extremo.

## 8. TDD

Separar lo determinista de lo no determinista:

| Capa | Estrategia de test |
|---|---|
| Parsers, normalización, `comparar_cifras` | Tests unitarios clásicos, muy numerosos |
| Herramientas MCP | Tests de contrato (esquemas) con APIs externas simuladas |
| Recuperación RAG | Índice de prueba con fixtures; la consulta debe devolver el fragmento esperado |
| Agentes ADK | Modelo simulado con respuestas fijas para probar flujo, llamadas a herramientas y errores |
| Calidad del LLM real | Conjunto de evaluación aparte, como métrica y no como test pass/fail |
| Frontend | Vitest + Playwright |

En la memoria se puede mostrar la evolución rojo-verde-refactor con commits claros y la cobertura alcanzada.

## 9. Evaluación

Crear un **conjunto de oro** de 60 a 100 afirmaciones con veredicto verificado manualmente y medir:

- **Extracción:** precisión, exhaustividad y F1.
- **Recuperación:** recall@k de la evidencia correcta.
- **Veredicto:** exactitud y F1 macro por clase.
- **Fidelidad de las citas:** porcentaje de afirmaciones respaldadas realmente por el fragmento citado.
- **Abstención:** si el sistema se abstiene cuando debe y solo entonces.
- **Coste y latencia** por ficha.

Comparativas posibles: LLM solo vs. RAG, con o sin herramienta de comparación determinista, recuperación vectorial vs. híbrida. Datasets como FEVER o las tareas CheckThat! de CLEF sirven de referencia metodológica, aunque conviene crear un conjunto propio en español.

## 10. Riesgos y mitigación

| Riesgo | Mitigación |
|---|---|
| Alucinación de citas | Validación automática de que cada cita existe en lo recuperado |
| Inyección de instrucciones vía documentos o páginas web | Tratar el contenido recuperado como datos, no como instrucciones; tests específicos |
| Fuentes desactualizadas o sesgadas | Metadatos de fecha y fiabilidad visibles en la ficha |
| Exceso de confianza | Mostrar siempre la confianza; revisión humana obligatoria |
| Derechos de autor | Limitarse a fuentes abiertas o material propio |

## 11. Plan de trabajo

1. **Cimientos:** esqueleto del proyecto, contenedores, CI con tests y TDD en los componentes deterministas.
2. **Datos y RAG:** corpus, indexación, búsqueda híbrida y su evaluación.
3. **Servidor MCP:** herramientas con contratos y tests.
4. **Agentes ADK:** primero extractor y juez, después el pipeline completo.
5. **Frontend:** vistas principales y flujo de revisión humana.
6. **Evaluación y ajuste:** conjunto de oro, métricas y comparativa de configuraciones.
7. **Memoria y defensa.**

**MVP realista:** extracción + búsqueda en datos oficiales + veredicto con citas + interfaz de revisión. Verificaciones previas, historial y exportación quedan como extras.

## 12. Estructura sugerida de la memoria

Introducción y motivación → estado del arte (fact-checking automatizado, RAG, agentes) → requisitos → arquitectura → implementación con TDD → evaluación y resultados → ética, límites y trabajo futuro.

