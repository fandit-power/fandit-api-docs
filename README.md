# API de FANDIT

Especificación OpenAPI y documentación técnica de la API REST de [FANDIT](https://fandit.es), la plataforma de búsqueda y gestión de subvenciones y ayudas públicas en España (BOE, diarios oficiales autonómicos y provinciales, BDNS y sedes electrónicas).

## Contenido de este repositorio

| Archivo | Descripción |
|---|---|
| [`openapi.yaml`](./openapi.yaml) / [`openapi.json`](./openapi.json) | Especificación OpenAPI 3.0.3 completa: 49 endpoints, parámetros, cuerpos de petición, respuestas y esquemas de autenticación. |
| [`llms-full.txt`](./llms-full.txt) | Documentación técnica completa en un único archivo continuo (autenticación, paginación, manejo de errores y referencia de cada endpoint con ejemplos reales de petición/respuesta). Pensado para ser leído de un tirón por un LLM o agente, sin depender de navegación entre páginas. |
| [`swagger-ui.html`](./swagger-ui.html) | Visor interactivo de la API (Swagger UI) con el spec ya incrustado. Disponible online en [https://fandit-power.github.io/fandit-api-docs/swagger-ui.html](https://fandit-power.github.io/fandit-api-docs/swagger-ui.html); también se puede abrir directamente haciendo doble clic, sin necesidad de servidor. |

## Empezando

1. Consigue tu API key desde tu cuenta de FANDIT.
2. Todas las peticiones van contra `https://api.fandit.es` (o `https://{tu-marca}.api.fandit.es` si tienes una instancia de marca blanca).
3. Autentícate con la cabecera `Authorization`, usando el prefijo que corresponda:
   - `Authorization: Token TU_API_KEY` — token de usuario, para el bloque de búsqueda y subvenciones (filtros, listado y detalle de convocatorias, concesiones, simuladores, normativa, evaluación, documentación requerida, relacionadas y chatbot).
   - `Authorization: ExpertToken TU_API_KEY` — token de experto, para el resto de endpoints de gestión (usuarios, clientes, contactos, expedientes).

Para explorar la API de forma interactiva, abre el [visor Swagger UI online](https://fandit-power.github.io/fandit-api-docs/swagger-ui.html) o `swagger-ui.html` en local. Para una referencia completa en texto (por ejemplo, para dársela como contexto a un LLM), usa `llms-full.txt`.

## Usar el spec OpenAPI

El `openapi.yaml` es compatible con las herramientas habituales del ecosistema OpenAPI:

```bash
# Importar en Postman: File → Import → openapi.yaml

# Generar un cliente (ejemplo con openapi-generator)
openapi-generator-cli generate -i openapi.yaml -g python -o ./cliente-fandit

# Validar el spec
npx @redocly/cli lint openapi.yaml
```

## Notas importantes

- **Créditos**: la mayoría de endpoints del bloque de subvenciones consumen créditos de la cuenta, además de requerir token. Un `403` puede significar token inválido o falta de créditos — son casos distintos.
- **Paginación**: el parámetro `page` siempre va como query param independiente, nunca dentro del JSON de `requestData`.
- **`requestData`**: varios endpoints (listado de subvenciones, concesiones, beneficiarios) reciben sus filtros serializados como JSON dentro de un único query param llamado `requestData`, en lugar de query params individuales.

Para el detalle completo de cada endpoint, parámetros y ejemplos reales, consulta [`llms-full.txt`](./llms-full.txt).

## Recursos relacionados

- [fandit-power.github.io/fandit-api-docs/swagger-ui.html](https://fandit-power.github.io/fandit-api-docs/swagger-ui.html) — visor interactivo de la API (Swagger UI).
- [fandit.es](https://fandit.es) — sitio principal.
- [fandit.es/llms.txt](https://fandit.es/llms.txt) — índice general del sitio para agentes/LLMs.
- [fandit.es/producto/integraciones-api](https://fandit.es/producto/integraciones-api) — información comercial sobre la API.

## Contribuir / reportar errores

Si detectas un error o una discrepancia entre esta documentación y el comportamiento real de la API, abre un issue en este repositorio.
