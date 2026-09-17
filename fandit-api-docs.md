# Documentación técnica de la API de FANDIT (referencia única)

Este documento reúne en un solo archivo continuo toda la documentación técnica de la API REST v2 de FANDIT (y los endpoints legacy v1 de Single Sign-On), pensado para ser leído de un tirón por un LLM sin depender de navegación entre páginas. Cubre: autenticación, manejo de errores, paginación, y la referencia completa de los 49 endpoints agrupados por bloque funcional (Filtros, Autenticación, Usuarios, Subvenciones, Expedientes, Clientes, Contactos, SSO legacy). No cubre otras áreas del producto FANDIT (Buscador web, Asistente IA, Alertas, Automatizaciones, etc.) más allá de lo que se expone vía esta API.

## URLs base

- Dominio genérico: `https://api.fandit.es`
- Marca blanca: `https://{plataforma}.api.fandit.es`, sustituyendo `{plataforma}` por el subdominio de tu marca (por ejemplo, `https://tu-marca.api.fandit.es/api/v2/`). Usa este dominio si tu cuenta está configurada como marca blanca; la URL debe contener el nombre de tu marca para que la API devuelva la información propia de esa marca en vez de la genérica.

Todos los endpoints de la versión 2 cuelgan de `/api/v2/`. Los endpoints legacy de Single Sign-On (sección "Autenticación (legacy / SSO)" más abajo) cuelgan de `/api/v1/users/partners-brand/` y solo están disponibles para marcas configuradas con SSO habilitado.

## Autenticación

La API usa dos tokens distintos según el bloque de endpoints, enviados en la cabecera `Authorization` con un prefijo distinto cada uno:

```
Authorization: Token TU_API_KEY          (token de usuario)
Authorization: ExpertToken TU_API_KEY    (token de experto)
```

- **Token de usuario** (prefijo `Token`): necesario para el bloque Buscador/Subvenciones completo (filtros generales, búsqueda y detalle de convocatorias, concesiones, simuladores de oportunidades, normativa, evaluación, documentación requerida, convocatorias relacionadas y chatbot) y para `GET /api/v2/users/current/`. Se obtiene mediante `POST /api/v2/users/login/` (email + contraseña).
- **Token de experto** (prefijo `ExpertToken`): necesario para el resto de endpoints de gestión — usuarios (alta/baja/edición), clientes, contactos y expedientes — y para `GET /api/v2/experts/current/`. Se obtiene mediante `POST /api/v2/experts/login` (email + contraseña).

Los propios endpoints de login (`/users/login/`, `/experts/login`) y los endpoints legacy de SSO en `/api/v1/users/partners-brand/` **no** llevan cabecera `Authorization`: en el caso de login porque es lo que se está solicitando, y en el caso de SSO porque el token (o un OTP) viaja como parámetro dentro del body de la propia petición, no como cabecera.

Dentro del bloque Subvenciones, la mayoría de endpoints además requieren **créditos disponibles** en la cuenta del usuario. Un `403` en esos endpoints puede significar dos cosas distintas y conviene distinguirlas de cara al usuario final: token ausente/incorrecto, o token válido pero sin créditos suficientes. El único endpoint del bloque Subvenciones que **no** consume créditos es `GET /api/v2/data-filters/`.

## Códigos de respuesta estándar

Salvo que se indique lo contrario en la ficha de un endpoint concreto, estos son los códigos que se pueden recibir en cualquier endpoint de la API:

- **200** — Todo correcto.
- **400** — Petición inválida: parámetros faltantes, con formato incorrecto, o que no pasan las validaciones del endpoint (ver notas específicas de cada endpoint; por ejemplo, rangos de fecha o de importe invertidos en el listado de subvenciones).
- **401** — La API key falta o no es válida. Revisa la cabecera `Authorization` y que lleve el prefijo `Token`.
- **403** — Dependiendo del endpoint: el token es del tipo que no corresponde (se necesita token de experto y se envió uno de usuario, o viceversa), o el token es válido pero la cuenta no tiene créditos suficientes para ese endpoint concreto.
- **404** — El recurso no existe. Ojo: en los endpoints de detalle de subvención (`fund-details`), un 404 no siempre significa que el id/slug esté mal escrito — esos endpoints solo cubren convocatorias **activas**; una convocatoria histórica/ya resuelta da 404 ahí aunque el identificador sea perfectamente válido (para esas, usar los endpoints de concesiones).
- **502** — Solo en `POST /api/v2/funds/chatbot/`: el servicio externo de chatbot no devolvió una respuesta JSON válida.


## Paginación

Todos los listados paginados de la API (incluidos los del bloque Subvenciones que usan `requestData`: `GET /funds/`, `GET /funds/concessions/`, `GET /funds/concessions/beneficiaries*`) usan el mismo esquema: el parámetro `page` va como query param independiente en la URL (empieza en `1`), **no** dentro del JSON de `requestData`. La respuesta trae `count`, `next`, `previous` y `results`.

`GET /funds/fund-related/{id}/` no está paginado: devuelve un array plano con todos los resultados.

Algunas convocatorias tienen más de 1.000 concesiones asociadas (por ejemplo, una edición de "Adelante Inversión" con 1.266). Si la pregunta que se está respondiendo requiere el total de resultados (importes agregados, conteos, "todas las empresas que..."), no basta con la primera página: hay que seguir pidiendo páginas siguientes usando `count` para saber cuántas faltan, hasta agotar los resultados.

## Otras particularidades a tener en cuenta

**Formato de fechas.** Los campos de fecha del bloque Subvenciones (`start_date`, `end_date`, `final_period_start_date`, `final_period_end_date`) deben enviarse en formato exacto `YYYY-MM-DD`.


**`bdns` debe ser un número entero.** El filtro `bdns` (código BDNS) del listado de subvenciones espera un valor numérico entero.


**Búsqueda semántica combinada con otros filtros.** `search_by_vectorized_text` se puede combinar con otros filtros (por ejemplo `is_open`) en la misma llamada; los filtros actúan de forma acumulativa. Con un texto muy largo o específico junto con filtros restrictivos, el resultado puede acotarse mucho e incluso llegar a 0 si ninguna convocatoria cumple todos los criterios a la vez — es el comportamiento esperado, no un error.


**`funds/concessions/` reutiliza el formato de `funds/` pero solo aplica un subconjunto de filtros.** `GET /funds/concessions/` acepta el mismo JSON `requestData` que `GET /funds/`, pero solo tienen efecto los campos indicados en la ficha del endpoint. Los campos no soportados ahí (`zip_code`, `status_code`, `minimis`, `min_budget`/`max_budget`, `search_by_vectorized_text`) se aceptan en el JSON pero no se aplican.


**`beneficiaries-by-cif` y `beneficiaries` devuelven listados distintos.** `GET /funds/concessions/beneficiaries-by-cif/` (filtra por `nif`) devuelve el listado de subvenciones/concesiones recibidas por una empresa concreta: una fila por cada convocatoria que esa empresa ha recibido. `GET /funds/concessions/beneficiaries/` (filtra por `fund_id`/`fund_slug`) devuelve el listado de beneficiarios de una convocatoria concreta: una fila por cada empresa que la ha recibido. Ambos resultados usan el mismo esquema de campos por fila (fund_id, fund_slug, fund_title, beneficiary_cif, beneficiary_name, concession_date, awarded_amount), pero no son intercambiables: cada uno solo acepta su propio parámetro de filtro.


## Referencia de endpoints

### Filtros

#### Filtros generales — `GET /api/v2/data-filters/`

**Autenticación:** Token de usuario


Devuelve el catálogo completo de valores de referencia usados por el resto de endpoints: tipos de solicitante, actividades/sectores, comunidades autónomas y provincias (con provinces anidadas), líneas de crédito, orígenes de fondos, tipos de ayuda, CNAEs, grupos de acción, etc. Cada elemento trae su id, que es el que hay que usar como filtro en los demás endpoints. Los ids devueltos aquí bajo applicants, activities y actions son compatibles con los campos applicants_v2/activities_v2/actions_v2 usados en /funds/ y /funds/concessions/ (mismo espacio de ids, a pesar de la diferencia de nombre). No consume créditos, solo requiere token de usuario.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/data-filters/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "action_items": [
    {
      "name": "Digitalización",
      "action_groups": [
        {
          "name": "Transformación digital",
          "action_items": [
            {
              "id": 1,
              "name": "Implantación de ERP"
            },
            {
              "id": 2,
              "name": "Comercio electrónico"
            }
          ]
        }
      ]
    }
  ],
  "activities": [
    {
      "id": 4,
      "name": "Industria manufacturera"
    },
    {
      "id": 6,
      "name": "Energías renovables"
    }
  ],
  "applicants": [
    {
      "id": 1,
      "name": "Autónomo"
    },
    {
      "id": 2,
      "name": "Pyme"
    }
  ],
  "cnaes": [
    {
      "code": "4321",
      "code2009": "4321",
      "id": 4321,
      "title": "Instalaciones eléctricas"
    }
  ],
  "communities": [
    {
      "action": "update",
      "code": "MD",
      "country": 1,
      "id": 13,
      "name": "Comunidad de Madrid",
      "provinces": [
        {
          "id": 28,
          "name": "Madrid"
        }
      ]
    }
  ],
  "credits": [
    {
      "description": "Línea ICO Empresas y Emprendedores",
      "id": 1,
      "name": "ICO Empresas"
    }
  ],
  "groups": [
    {
      "id": 1,
      "name": "Administradores"
    },
    {
      "id": 2,
      "name": "Gestores"
    }
  ],
  "origins": [
    {
      "action": "update",
      "id": 1,
      "name": "Fondos Next Generation EU"
    }
  ],
  "provinces": [
    {
      "id": 28,
      "name": "Madrid"
    },
    {
      "id": 46,
      "name": "Valencia"
    }
  ],
  "regions_types": [
    {
      "code": 1,
      "id": 1,
      "name": "Objetivo Transición Justa"
    }
  ],
  "types_fund": [
    {
      "id": 1,
      "name": "Subvención"
    },
    {
      "id": 2,
      "name": "Préstamo"
    }
  ],
  "user_profile": [
    {
      "id": 1,
      "name": "Básico"
    },
    {
      "id": 2,
      "name": "Pro"
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


### Autenticación

#### Current de experto — `GET /api/v2/experts/current/`

**Autenticación:** Token de experto


Petición para obtener toda la información del expert logueado.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/experts/current/' \
  --header 'Authorization: ExpertToken TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 542,
  "first_name": "Carlos",
  "last_name": "Ruiz",
  "username": "carlos.ruiz",
  "phone": "+34600000005",
  "business_name": "Consultora Ayudas y Subvenciones S.L.",
  "profile_avatar": "https://fandit-media.s3.amazonaws.com/experts/avatars/542.jpg",
  "profile_avatar_thumbnail": "https://fandit-media.s3.amazonaws.com/experts/avatars/542_thumb.jpg",
  "communities": [
    {
      "id": 1,
      "name": "Andalucía"
    },
    {
      "id": 13,
      "name": "Madrid"
    }
  ],
  "provinces": [
    {
      "id": 28,
      "name": "Madrid"
    },
    {
      "id": 41,
      "name": "Sevilla"
    }
  ],
  "actions": [
    {
      "id": 3,
      "name": "Digitalización"
    },
    {
      "id": 7,
      "name": "Internacionalización"
    }
  ],
  "action_items": [
    {
      "id": 12,
      "name": "Implantación de ERP"
    }
  ],
  "applicant_types": [
    {
      "id": 2,
      "name": "Pyme"
    }
  ],
  "fund_types": [
    {
      "id": 1,
      "name": "Subvención"
    },
    {
      "id": 4,
      "name": "Préstamo"
    }
  ],
  "region_types": [
    {
      "id": 1,
      "name": "Nacional"
    }
  ],
  "activities": [
    {
      "id": 15,
      "name": "Industria"
    }
  ],
  "expert_types": [
    {
      "id": 2,
      "name": "Consultoría"
    }
  ],
  "solution_types": [
    {
      "id": 1,
      "name": "Gestión de ayudas"
    }
  ],
  "is_super_expert": true,
  "groups": [
    "Expert"
  ],
  "permissions": [],
  "active_custom_forms": 4,
  "marketplace_membership": {
    "id": 2,
    "name": "Premium",
    "funds_limit": 50,
    "leads_limit": 20,
    "leads_etg": true,
    "priority": 1
  },
  "integrations": [
    {
      "id": 9,
      "extension_name": "CRM Sync",
      "handler_name": "crm_sync_handler",
      "url_api": null,
      "created_at": "2025-02-10T09:30:00Z",
      "updated_at": "2026-06-15T11:12:00Z",
      "is_active": true,
      "extension": 3
    }
  ],
  "platforms_with_billing_access": [
    "FANDIT"
  ],
  "marketplace_partners_info": [
    {
      "name": "FANDIT Marketplace",
      "schema": "https",
      "url": "marketplace.fandit.es"
    }
  ],
  "email": "carlos.ruiz@example.com",
  "website": "https://consultora-ayudas.es",
  "contact_email": "contacto@example.com",
  "profile_image": "https://fandit-media.s3.amazonaws.com/experts/profile/542.jpg",
  "title": "Consultor senior en subvenciones públicas",
  "experience": "Más de 10 años gestionando ayudas y subvenciones para pymes.",
  "services": "Diagnóstico de ayudas, tramitación y justificación de subvenciones.",
  "areas": "Digitalización, Internacionalización, I+D+i",
  "extra_info": "Colegiado en el Colegio de Economistas de Madrid.",
  "marketplace_visibility": true,
  "marketplace_name": "Consultora Ayudas y Subvenciones",
  "slug": "consultora-ayudas-y-subvenciones",
  "assessment": 5,
  "status": 1,
  "daily_downloaded_instructions": [
    12,
    8,
    15
  ],
  "monthly_downloaded_instructions": [
    340,
    298
  ],
  "custom_forms_available": 6,
  "access_ecosystem": 1,
  "partners_with_user": [
    "FANDIT"
  ],
  "workflow_funds": [
    101,
    102,
    205
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Login de experto — `POST /api/v2/experts/login`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para obtener el token de experto.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo electrónico con el que registrar el usuario. |
| `password` | string | Sí | Contraseña. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/experts/login' \
  --data '{"email":"tu@example.com","password":"TU_CONTRASEÑA"}' \
  --header 'Content-Type: application/json'
```


**Ejemplo de respuesta:**


```json
{
  "token": "EXPERT_TOKEN_DE_EJEMPLO",
  "expert": {
    "id": 542,
    "first_name": "Carlos",
    "last_name": "Ruiz",
    "username": "carlos.ruiz",
    "phone": "+34600000005",
    "business_name": "Consultora Ayudas y Subvenciones S.L.",
    "profile_avatar": "https://fandit-media.s3.amazonaws.com/experts/avatars/542.jpg",
    "profile_avatar_thumbnail": "https://fandit-media.s3.amazonaws.com/experts/avatars/542_thumb.jpg",
    "communities": [
      {
        "id": 1,
        "name": "Andalucía"
      },
      {
        "id": 13,
        "name": "Madrid"
      }
    ],
    "provinces": [
      {
        "id": 28,
        "name": "Madrid"
      },
      {
        "id": 41,
        "name": "Sevilla"
      }
    ],
    "actions": [
      {
        "id": 3,
        "name": "Digitalización"
      },
      {
        "id": 7,
        "name": "Internacionalización"
      }
    ],
    "action_items": [
      {
        "id": 12,
        "name": "Implantación de ERP"
      }
    ],
    "applicant_types": [
      {
        "id": 2,
        "name": "Pyme"
      }
    ],
    "fund_types": [
      {
        "id": 1,
        "name": "Subvención"
      },
      {
        "id": 4,
        "name": "Préstamo"
      }
    ],
    "region_types": [
      {
        "id": 1,
        "name": "Nacional"
      }
    ],
    "activities": [
      {
        "id": 15,
        "name": "Industria"
      }
    ],
    "expert_types": [
      {
        "id": 2,
        "name": "Consultoría"
      }
    ],
    "solution_types": [
      {
        "id": 1,
        "name": "Gestión de ayudas"
      }
    ],
    "is_super_expert": true,
    "groups": [
      "Expert"
    ],
    "permissions": [],
    "active_custom_forms": 4,
    "marketplace_membership": {
      "id": 2,
      "name": "Premium",
      "funds_limit": 50,
      "leads_limit": 20,
      "leads_etg": true,
      "priority": 1
    },
    "integrations": [
      {
        "id": 9,
        "extension_name": "CRM Sync",
        "handler_name": "crm_sync_handler",
        "url_api": null,
        "created_at": "2025-02-10T09:30:00Z",
        "updated_at": "2026-06-15T11:12:00Z",
        "is_active": true,
        "extension": 3
      }
    ],
    "platforms_with_billing_access": [
      "FANDIT"
    ],
    "marketplace_partners_info": [
      {
        "name": "FANDIT Marketplace",
        "schema": "https",
        "url": "marketplace.fandit.es"
      }
    ],
    "email": "carlos.ruiz@example.com",
    "website": "https://consultora-ayudas.es",
    "contact_email": "contacto@example.com",
    "profile_image": "https://fandit-media.s3.amazonaws.com/experts/profile/542.jpg",
    "title": "Consultor senior en subvenciones públicas",
    "experience": "Más de 10 años gestionando ayudas y subvenciones para pymes.",
    "services": "Diagnóstico de ayudas, tramitación y justificación de subvenciones.",
    "areas": "Digitalización, Internacionalización, I+D+i",
    "extra_info": "Colegiado en el Colegio de Economistas de Madrid.",
    "marketplace_visibility": true,
    "marketplace_name": "Consultora Ayudas y Subvenciones",
    "slug": "consultora-ayudas-y-subvenciones",
    "assessment": 5,
    "status": 1,
    "daily_downloaded_instructions": [
      12,
      8,
      15
    ],
    "monthly_downloaded_instructions": [
      340,
      298
    ],
    "custom_forms_available": 6,
    "access_ecosystem": 1,
    "partners_with_user": [
      "FANDIT"
    ],
    "workflow_funds": [
      101,
      102,
      205
    ]
  }
}
```


**Notas de errores específicas de este endpoint:**




---


#### Current de usuario — `GET /api/v2/users/current/`

**Autenticación:** Token de usuario


Petición para obtener toda la información del usuario logueado.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/users/current/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 10482,
  "email": "laura.gomez@example.com",
  "username": "laura.gomez",
  "first_name": "Laura",
  "last_name": "Gómez",
  "platform": 3,
  "is_active": true,
  "communities_list": [
    1,
    8
  ],
  "provinces_list": [
    28,
    41
  ],
  "applicants_list": [
    2,
    5
  ],
  "region_types_list": [
    1
  ],
  "actions_list": [
    3,
    7,
    12
  ],
  "activities_list": [
    15,
    22
  ],
  "general_notifications": true,
  "distributor": "FANDIT",
  "business_name": "Innovatech Soluciones S.L.",
  "phone": "+34600000000",
  "fund_types_list": [
    1,
    4
  ],
  "origins_list": [
    2
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Login de usuario — `POST /api/v2/users/login/`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para obtener los token de usuario y experto si es el caso.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo electrónico con el que registrar el usuario. |
| `password` | string | Sí | Contraseña. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/users/login/' \
  --data '{"email":"tu@example.com","password":"TU_CONTRASEÑA"}' \
  --header 'Content-Type: application/json'
```


**Ejemplo de respuesta:**


```json
{
  "token": "TOKEN_DE_EJEMPLO",
  "ExpertToken": "EXPERT_TOKEN_DE_EJEMPLO",
  "user": {
    "id": 10482,
    "email": "laura.gomez@example.com",
    "username": "laura.gomez",
    "first_name": "Laura",
    "last_name": "Gómez",
    "platform": 3,
    "is_active": true,
    "communities_list": [
      1,
      8
    ],
    "provinces_list": [
      28,
      41
    ],
    "applicants_list": [
      2,
      5
    ],
    "region_types_list": [
      1
    ],
    "actions_list": [
      3,
      7,
      12
    ],
    "activities_list": [
      15,
      22
    ],
    "general_notifications": true,
    "distributor": "FANDIT",
    "business_name": "Innovatech Soluciones S.L.",
    "phone": "+34600000000",
    "fund_types_list": [
      1,
      4
    ],
    "origins_list": [
      2
    ]
  }
}
```


**Notas de errores específicas de este endpoint:**




---


### Autenticación (legacy / SSO)

#### Validar correo — `GET /api/v1/users/partners-brand/check-user/{email}`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para validar si un correo ya existe en la plataforma.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `email` | path | string | Sí | Correo electrónico a validar. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v1/users/partners-brand/check-user/laura.gomez@example.com'
```


**Ejemplo de respuesta:**


```json
{
  "username": "laura.gomez"
}
```


**Notas de errores específicas de este endpoint:**




---


#### Eliminar de usuario — `DELETE /api/v1/users/partners-brand/delete-user/`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para eliminar un usuario registrado.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `token` | query | string | Sí | OTP del usuario. |


**Ejemplo de petición:**


```bash
curl --request DELETE \
  --url 'https://api.fandit.es/api/v1/users/partners-brand/delete-user/' \
  --data '{"token":"927154"}' \
  --header 'Content-Type: application/json'
```


**Ejemplo de respuesta:**


```json
"Usuario eliminado correctamente."
```


**Notas de errores específicas de este endpoint:**




---


#### Login de usuario — `POST /api/v1/users/partners-brand/login/`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para iniciar sesión de un usuario con su correo y contraseña.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo electrónico con el que registrar el usuario. |
| `password` | string | Sí | Contraseña. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v1/users/partners-brand/login/' \
  --data '{"email":"laura.gomez@example.com","password":"TU_CONTRASEÑA"}' \
  --header 'Content-Type: application/json'
```


**Ejemplo de respuesta:**


```json
{
  "otp": "482913",
  "uuid": "3f2504e0-4f89-11d3-9a0c-0305e82c3301"
}
```


**Notas de errores específicas de este endpoint:**




---


#### Cambio de contraseña — `PUT /api/v1/users/partners-brand/password-change/`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para cambiar la contraseña de un usuario ya registrado.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `token` | string | Sí | Token de usuario. |
| `password` | string | Sí | Contraseña. |
| `password_confirm` | string | Sí | Confirmación de contraseña. |


**Ejemplo de petición:**


```bash
curl --request PUT \
  --url 'https://api.fandit.es/api/v1/users/partners-brand/password-change/' \
  --data '{"token":"TU_TOKEN","password":"TU_NUEVA_CONTRASEÑA","password_confirm":"TU_NUEVA_CONTRASEÑA"}' \
  --header 'Content-Type: application/json'
```


**Ejemplo de respuesta:**


```json
"Contraseña actualizada correctamente."
```


**Notas de errores específicas de este endpoint:**




---


#### Registro de usuarios — `POST /api/v1/users/partners-brand/registration/`

**Autenticación:** Ninguna (login/registro; el token se obtiene o viaja en el body)


Petición para registrar un nuevo usuario en la plataforma.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `first_name` | string | Sí | Nombre del usuario. |
| `last_name` | string | No | Apellido del usuario. |
| `email` | string | Sí | Correo electrónico con el que registrar el usuario. |
| `password1` | string | Sí | Contraseña. |
| `password2` | string | Sí | Confirmación de contraseña. |
| `platform` | string | No | Plataforma hija, en caso de tener. |
| `general_notifications` | boolean | No | Recibir notificaciones generales. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v1/users/partners-brand/registration/' \
  --data '{"first_name":"Laura","last_name":"Gómez","email":"laura.gomez@example.com","password1":"TU_CONTRASEÑA","password2":"TU_CONTRASEÑA","platform":"","general_notifications":false}' \
  --header 'Content-Type: application/json'
```


**Ejemplo de respuesta:**


```json
{
  "otp": "927154",
  "uuid": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```


**Notas de errores específicas de este endpoint:**




---


### Usuarios

#### Listado de usuarios — `GET /api/v2/users/`

**Autenticación:** Token de experto


Petición paginada para obtener todos los usuarios registrados.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `page` | query | number | Sí | Página del listado a visualizar. |
| `page_size` | query | number | Sí | Tamaño de la paginación. |
| `platform` | query | string | No | Marca gris seleccionada. |
| `requestData` | query | object | No | Filtros adicionales de búsqueda, serializados como JSON en un único query param. |

Campos de `requestData`:

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `order` | string | No | Nombre de atributo por el que ordenar los resultados. |
| `general_text` | string | No | Texto a buscar en todas las columnas. |
| `user` | string | No | Correo electrónico o nombre del usuario a filtrar. |
| `profile` | string | No | Perfil de los usuarios a filtrar (Básico, Pro, Equipo). |
| `start_date` | string | No | Fecha de creación (inicio de rango). |
| `end_date` | string | No | Fecha de creación (fin de rango). |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/users/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'page_size=20' \
  --data-urlencode 'requestData={"order":"-date_joined"}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 10482,
      "email": "usuario@example.com",
      "username": "usuario_empresa",
      "first_name": "Laura",
      "last_name": "Gómez",
      "platform": "fandit",
      "is_active": true,
      "communities_list": [
        1,
        8
      ],
      "provinces_list": [
        28,
        8
      ],
      "applicants_list": [
        1,
        2
      ],
      "region_types_list": [
        1
      ],
      "actions_list": [
        1,
        2,
        3
      ],
      "activities_list": [
        4
      ],
      "general_notifications": true,
      "distributor": "FANDIT",
      "business_name": "Empresa Ejemplo SL",
      "phone": "600000000",
      "fund_types_list": [
        1,
        2
      ],
      "origins_list": [
        1
      ],
      "groups_data": [
        {
          "id": 1,
          "name": "Administradores"
        },
        {
          "id": 2,
          "name": "Gestores"
        }
      ]
    },
    {
      "id": 10501,
      "email": "contacto@example.com",
      "username": "innovatech_sl",
      "first_name": "Carlos",
      "last_name": "Fernández",
      "platform": "fandit",
      "is_active": true,
      "communities_list": [
        10
      ],
      "provinces_list": [
        46
      ],
      "applicants_list": [
        1
      ],
      "region_types_list": [
        2
      ],
      "actions_list": [
        2
      ],
      "activities_list": [
        6
      ],
      "general_notifications": false,
      "distributor": "FANDIT",
      "business_name": "Innovatech Soluciones SL",
      "phone": "600000004",
      "fund_types_list": [
        3
      ],
      "origins_list": [
        2
      ],
      "groups_data": [
        {
          "id": 2,
          "name": "Gestores"
        }
      ]
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Crear usuario — `POST /api/v2/users/`

**Autenticación:** Token de experto


Petición para crear un nuevo usuario.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo electrónico con el que registrar el usuario. |
| `password` | string | Sí | Contraseña. |
| `username` | string | Sí | Username del usuario. |
| `first_name` | string | Sí | Nombre del usuario. |
| `last_name` | string | No | Apellido del usuario. |
| `business_name` | string | No | Nombre o razón social. |
| `phone` | string | No | Número telefónico del usuario. |
| `distributor` | string | No | Nombre del distribuidor. |
| `general_notifications` | boolean | No | Recibir notificaciones generales. |
| `communities` | array[integer] | No | Comunidades de interes del usuario. |
| `provinces` | array[integer] | No | Provincias de interes del usuario. |
| `applicants` | array[integer] | No | Tipo de solicitante del usuario. |
| `region_types` | array[integer] | No | Regiones de interes del usuario. |
| `action_items` | array[integer] | No | Acción a llevar a cabo por el usuario. |
| `activities` | array[integer] | No | Sector económico del usuario. |
| `fund_types` | array[integer] | No | Tipo de ayuda. |
| `origins` | array[integer] | No | Origen de los fondos. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/users/' \
  --data '{"email":"nuevo.usuario@example.com","password":"TU_CONTRASEÑA","username":"nuevo_usuario","first_name":"Marta","last_name":"Ruiz","business_name":"Consultora Ejemplo SL","phone":"600000003","distributor":"FANDIT","general_notifications":true,"communities":[1],"provinces":[28],"applicants":[1],"region_types":[1],"action_items":[1],"activities":[3],"fund_types":[1],"origins":[1]}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 10530,
  "email": "nuevo.usuario@example.com",
  "username": "nuevo_usuario",
  "first_name": "Marta",
  "last_name": "Ruiz",
  "platform": "fandit",
  "is_active": true,
  "general_notifications": true,
  "distributor": "FANDIT",
  "business_name": "Consultora Ejemplo SL",
  "phone": "600000003",
  "communities_list": [
    1
  ],
  "provinces_list": [
    28
  ],
  "applicants_list": [
    1
  ],
  "region_types_list": [
    1
  ],
  "action_items_list": [
    1
  ],
  "activities_list": [
    3
  ],
  "fund_types_list": [
    1
  ],
  "origins_list": [
    1
  ],
  "groups_data": [
    {
      "id": 2,
      "name": "Gestores"
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Eliminar usuario — `DELETE /api/v2/users/{id}/`

**Autenticación:** Token de experto


Petición para eliminar un usuario específico.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | string | Sí | Identificador del usuario a buscar. |


**Ejemplo de petición:**


```bash
curl --request DELETE \
  --url 'https://api.fandit.es/api/v2/users/10482/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
"User deleted successfully"
```


**Notas de errores específicas de este endpoint:**




---


#### Detalles del usuario — `GET /api/v2/users/{id}/`

**Autenticación:** Token de experto


Petición para obtener toda la información de un usuario usuario específico.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | number | Sí | Identificador del usuario a buscar. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/users/10482/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 10482,
  "email": "usuario@example.com",
  "username": "usuario_empresa",
  "first_name": "Laura",
  "last_name": "Gómez",
  "platform": "fandit",
  "is_active": true,
  "general_notifications": true,
  "distributor": "FANDIT",
  "business_name": "Empresa Ejemplo SL",
  "phone": "600000000",
  "communities_list": [
    1,
    8
  ],
  "provinces_list": [
    28,
    8
  ],
  "applicants_list": [
    1,
    2
  ],
  "region_types_list": [
    1
  ],
  "action_items_list": [
    1,
    2,
    3
  ],
  "activities_list": [
    4
  ],
  "fund_types_list": [
    1,
    2
  ],
  "origins_list": [
    1
  ],
  "groups_data": [
    {
      "id": 1,
      "name": "Administradores"
    },
    {
      "id": 2,
      "name": "Gestores"
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Actualizar usuario — `PATCH /api/v2/users/{id}/`

**Autenticación:** Token de experto


Petición para actualizar los datos de un usuario específico. `PUT` también está soportado en esta misma URL y se comporta de forma idéntica a `PATCH` (actualización parcial en ambos casos); se documenta solo `PATCH` porque es el método recomendado.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | string | Sí | Identificador del usuario a buscar. |


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `password` | string | No | Contraseña. |
| `username` | string | No | Username del usuario. |
| `first_name` | string | No | Nombre del usuario. |
| `last_name` | string | No | Apellido del usuario. |
| `business_name` | string | No | Nombre o razón social. |
| `phone` | string | No | Número telefónico del usuario. |
| `distributor` | string | No | Nombre del distribuidor. |
| `general_notifications` | boolean | No | Recibir notificaciones generales. |
| `communities` | array[integer] | No | Comunidades de interes del usuario. |
| `provinces` | array[integer] | No | Provincias de interes del usuario. |
| `applicants` | array[integer] | No | Tipo de solicitante del usuario. |
| `region_types` | array[integer] | No | Regiones de interes del usuario. |
| `action_items` | array[integer] | No | Acción a llevar a cabo por el usuario. |
| `activities` | array[integer] | No | Sector económico del usuario. |
| `fund_types` | array[integer] | No | Tipo de ayuda. |
| `origins` | array[integer] | No | Origen de los fondos. |


**Ejemplo de petición:**


```bash
curl --request PATCH \
  --url 'https://api.fandit.es/api/v2/users/10482/' \
  --data '{"phone":"600000000","business_name":"Empresa Ejemplo SL","general_notifications":true}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 10482,
  "email": "usuario@example.com",
  "username": "usuario_empresa",
  "first_name": "Laura",
  "last_name": "Gómez",
  "platform": "fandit",
  "is_active": true,
  "general_notifications": true,
  "distributor": "FANDIT",
  "business_name": "Empresa Ejemplo SL",
  "phone": "600000000",
  "communities_list": [
    1,
    8
  ],
  "provinces_list": [
    28,
    8
  ],
  "applicants_list": [
    1,
    2
  ],
  "region_types_list": [
    1
  ],
  "action_items_list": [
    1,
    2,
    3
  ],
  "activities_list": [
    4
  ],
  "fund_types_list": [
    1,
    2
  ],
  "origins_list": [
    1
  ],
  "groups_data": [
    {
      "id": 1,
      "name": "Administradores"
    },
    {
      "id": 2,
      "name": "Gestores"
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Estadísticas de uso — `GET /api/v2/users/usage-statistics/`

**Autenticación:** Token de usuario o token de experto


Petición para obtener el consumo de créditos del usuario o experto autenticado en el mes en curso: créditos usados, límite mensual, bolsa de créditos extra disponible y fecha del próximo reinicio.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/users/usage-statistics/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "user_id": 10482,
  "user_email": "laura.gomez@example.com",
  "monthly_used_credits": 34,
  "max_credits": 100,
  "credits_bag": 10,
  "credits_reset_date": "2026-09-01"
}
```


**Notas de errores específicas de este endpoint:**




---


### Subvenciones

#### Detalles de una subvención — `GET /api/v2/fund-details/{identifier}/`

**Autenticación:** Token de usuario


Detalle completo de una convocatoria activa. Acepta como identificador el id numérico o el slug de la convocatoria (el slug se normaliza a minúsculas). Este endpoint solo cubre convocatorias activas: una convocatoria histórica o ya resuelta devuelve 404 aunque el identificador sea válido; para consultar esas, usa /funds/concessions/ o /funds/concessions/beneficiaries*. Para preguntas sobre cómo aumentar las probabilidades de éxito de una solicitud, combina este endpoint con fund-evaluation y fund-required-documents. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `identifier` | path | string | Sí | ID numérico o slug de la convocatoria. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/fund-details/kit-digital-segmento-iii/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "slug": "kit-digital-segmento-iii",
  "formatted_title": "Kit Digital - Segmento III (0-2 empleados)",
  "goal_extra": "Fomentar la digitalización de pequeñas empresas y autónomos mediante la adopción de soluciones de digitalización disponibles en el mercado.",
  "status_text": "Abierta",
  "scope": "Nacional",
  "total_amount": 500000000,
  "request_amount": 2000,
  "publisher": "Red.es",
  "applicants": "Autónomos y microempresas de 0 a 2 empleados",
  "term": "Hasta agotar presupuesto o el 31/12/2026",
  "help_type": "Subvención directa",
  "expenses": "Servicios de digitalización prestados por Agentes Digitalizadores adheridos",
  "fund_execution_period": "12 meses desde la fecha de concesión",
  "line": "Programa Kit Digital",
  "extra_limit": 2000,
  "info_extra": "Convocatoria financiada por el Plan de Recuperación, Transformación y Resiliencia - Next Generation EU"
}
```


**Notas de errores específicas de este endpoint:**


- `404`: No existe o no está activa (ver nota en la descripción sobre convocatorias históricas).



---


#### Listado de subvenciones — `GET /api/v2/funds/`

**Autenticación:** Token de usuario


Listado de convocatorias activas (active=True) con un amplio sistema de filtros, incluyendo búsqueda semántica vectorizada (search_by_vectorized_text) que puede autocompletar filtros. Si se necesita el total de resultados y la respuesta no cabe en una página, seguir pidiendo páginas siguientes con el parámetro page hasta agotar los resultados (usar count). Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `requestData` | query | object | No | Objeto de filtros serializado como JSON. Ejemplo: {"provinces":[33],"is_open":true,"sizes":[2],"order":"-total_amount"} |
| `page` | query | integer | No | Página del listado. Es un parámetro de query independiente, no va dentro de requestData. |

Campos de `requestData` (Todos los campos son opcionales. Se envía serializado como JSON dentro del query param `requestData` (no como query params individuales). Si `requestData` no se envía, se aplican los valores por defecto (equivale a una búsqueda sin filtros).):

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `provinces` | array[integer] | No | IDs de provincia. |
| `communities` | array[integer] | No | IDs de comunidad autónoma. |
| `types` | array[integer] | No | IDs de tipo de convocatoria. |
| `region_types` | array[integer] | No | IDs de ámbito territorial. |
| `applicants_v2` | array[integer] | No | IDs de tipo de solicitante (mismo espacio de ids que `applicants` en /data-filters/). |
| `actions_v2` | array[integer] | No | IDs de acción/línea de ayuda (mismo espacio de ids que `actions` en /data-filters/). |
| `activities_v2` | array[integer] | No | IDs de actividad/sector (mismo espacio de ids que `activities` en /data-filters/). |
| `origins` | array[integer] | No | IDs de origen de fondos. |
| `credit_types` | array[integer] | No | IDs de tipo de ayuda. |
| `sizes` | array[integer] | No | IDs de tamaño de empresa. |
| `profiles` | array[integer] | No | IDs de perfil de solicitante. |
| `search_by_text` | string | No | Búsqueda de texto literal (coincidencia de palabras en el título). Úsalo para nombres concretos de convocatoria. |
| `search_by_vectorized_text` | string | No | Búsqueda semántica vía embeddings: encuentra convocatorias relacionadas con una idea aunque no compartan las palabras exactas. Puede autocompletar automáticamente applicants_v2/actions_v2/communities/provinces si vienen vacíos. Al combinarlo con otros filtros estructurados (por ejemplo `is_open`), estos actúan de forma acumulativa: cuanto más específico sea el texto y más restrictivos los filtros adicionales, menos resultados devolverá la búsqueda, pudiendo llegar a cero si ninguna convocatoria cumple todos los criterios a la vez. |
| `is_open` | boolean | No | Si se indica, filtra solo convocatorias abiertas (true) o no abiertas (false). Para distinguir entre pendientes y cerradas usa `status_code`. |
| `reviewed` | boolean | No | Acepta true/"true"/1/"1" como verdadero; cualquier otro valor se trata como falso. |
| `start_date` | string | No | Fecha de apertura, inicio de rango. Formato YYYY-MM-DD. |
| `end_date` | string | No | Fecha de apertura, fin de rango. Formato YYYY-MM-DD. |
| `final_period_start_date` | string | No | Fecha de cierre, inicio de rango. Formato YYYY-MM-DD. |
| `final_period_end_date` | string | No | Fecha de cierre, fin de rango. Formato YYYY-MM-DD. |
| `platform` | string | No | Slug de plataforma. |
| `office` | - | No | Filtro de oficina/organismo (int o string). |
| `bdns` | integer | No | Código BDNS. Debe enviarse como número entero. |
| `min_budget` | number | No | Presupuesto de la ayuda, mínimo de rango. |
| `max_budget` | number | No | Presupuesto de la ayuda, máximo de rango. |
| `order` | string | No | Atributo por el que ordenar los resultados. Antepón un guion (-) para orden descendente. Valores admitidos: `order_score` (afinidad/relevancia con la búsqueda, valor por defecto), `total_amount` (presupuesto), `start_date`, `end_date`, `final_period_start_date` y `final_period_end_date` (fechas). Ejemplos: `-total_amount` (mayor presupuesto primero), `-end_date` (cierre más próximo primero). |
| `zip_code` | string | No | Código postal por el que filtrar. |
| `status_code` | integer | No | Filtro por estado de la convocatoria: `0` = pendiente (aún no abierta), `1` = abierta, `2` = cerrada. Valores: 0, 1, 2. |
| `minimis` | boolean | No | Filtra por régimen de minimis. |
| `min_total_amount` | number | No | Importe total de la convocatoria, mínimo de rango. |
| `max_total_amount` | number | No | Importe total de la convocatoria, máximo de rango. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'requestData={"provinces":[33],"is_open":true,"sizes":[2],"order":"-total_amount"}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "count": 245,
  "next": "https://api.fandit.es/api/v2/funds/?page=2&requestData=...",
  "previous": null,
  "results": [
    {
      "id": 1,
      "slug": "kit-consulting-2026",
      "formatted_title": "Kit Consulting",
      "status_text": "Abierta",
      "new_entity": 1,
      "total_amount": 12000
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**


- `400`: Filtros de fecha/monto inválidos, p.ej. {"errors": "Rango de fechas inválidas"} o {"errors": "Rango de montos inválidos"}.



---


#### Chatbot de subvención — `POST /api/v2/funds/chatbot/`

**Autenticación:** Token de usuario


Proxy hacia un servicio externo de chatbot, con RAG acotado por fund_id. Diseño recomendado: stateless — reenviar el historial completo (record) en cada llamada, sin que Fandit guarde sesión en servidor. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `fund_id` | integer | Sí | Id de la convocatoria (404 si no existe). |
| `prompt` | string | Sí | Pregunta del usuario. |
| `record` | array[object] | No | Historial de conversación. |
| `max_tokens` | integer | No | Límite de tokens de la respuesta. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/funds/chatbot/' \
  --data '{"fund_id":5891,"prompt":"¿Cuál es el plazo de solicitud?","record":[],"max_tokens":2048}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "response": "El plazo de solicitud para el Kit Digital - Segmento III permanece abierto hasta agotar el presupuesto asignado o, como máximo, hasta el 31 de diciembre de 2026. Se recomienda presentar la solicitud lo antes posible, ya que la concesión se realiza por estricto orden de entrada."
}
```


**Notas de errores específicas de este endpoint:**


- `400`: Faltan campos o tipos inválidos.

- `404`: fund_id inexistente.

- `502`: El servicio externo de chatbot no devolvió JSON válido.



---


#### Listado de concesiones — `GET /api/v2/funds/concessions/`

**Autenticación:** Token de usuario


Listado de fondos con concesiones publicadas (with_concessions=True): ayudas ya resueltas, no convocatorias abiertas en general. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `requestData` | query | object | No | Mismo formato que en /funds/, pero solo un subconjunto de los campos tiene efecto aquí (ver descripción de cada campo). |
| `page` | query | integer | No | Página del listado. Es un parámetro de query independiente, no va dentro de requestData. |

Campos de `requestData` (Todos los campos son opcionales. Se envía serializado como JSON dentro del query param `requestData` (no como query params individuales). Si `requestData` no se envía, se aplican los valores por defecto (equivale a una búsqueda sin filtros). Este endpoint reutiliza el mismo formato requestData que /funds/, pero solo tienen efecto los siguientes campos: actions_v2, activities_v2, applicants_v2, bdns, communities, credit_types, end_date, final_period_end_date, final_period_start_date, max_total_amount, min_total_amount, office, origins, profiles, provinces, region_types, reviewed, search_by_text, sizes, start_date, types. El resto de campos (zip_code, status_code, minimis, min_budget/max_budget, search_by_vectorized_text) se acepta en el JSON pero no se aplica en este endpoint.):

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `provinces` | array[integer] | No | IDs de provincia. |
| `communities` | array[integer] | No | IDs de comunidad autónoma. |
| `types` | array[integer] | No | IDs de tipo de convocatoria. |
| `region_types` | array[integer] | No | IDs de ámbito territorial. |
| `applicants_v2` | array[integer] | No | IDs de tipo de solicitante (mismo espacio de ids que `applicants` en /data-filters/). |
| `actions_v2` | array[integer] | No | IDs de acción/línea de ayuda (mismo espacio de ids que `actions` en /data-filters/). |
| `activities_v2` | array[integer] | No | IDs de actividad/sector (mismo espacio de ids que `activities` en /data-filters/). |
| `origins` | array[integer] | No | IDs de origen de fondos. |
| `credit_types` | array[integer] | No | IDs de tipo de ayuda. |
| `sizes` | array[integer] | No | IDs de tamaño de empresa. |
| `profiles` | array[integer] | No | IDs de perfil de solicitante. |
| `search_by_text` | string | No | Búsqueda de texto literal (coincidencia de palabras en el título). Úsalo para nombres concretos de convocatoria. |
| `search_by_vectorized_text` | string | No | Búsqueda semántica vía embeddings: encuentra convocatorias relacionadas con una idea aunque no compartan las palabras exactas. Puede autocompletar automáticamente applicants_v2/actions_v2/communities/provinces si vienen vacíos. Al combinarlo con otros filtros estructurados (por ejemplo `is_open`), estos actúan de forma acumulativa: cuanto más específico sea el texto y más restrictivos los filtros adicionales, menos resultados devolverá la búsqueda, pudiendo llegar a cero si ninguna convocatoria cumple todos los criterios a la vez. |
| `is_open` | boolean | No | Si se indica, filtra solo convocatorias abiertas (true) o no abiertas (false). Para distinguir entre pendientes y cerradas usa `status_code`. |
| `reviewed` | boolean | No | Acepta true/"true"/1/"1" como verdadero; cualquier otro valor se trata como falso. |
| `start_date` | string | No | Fecha de apertura, inicio de rango. Formato YYYY-MM-DD. |
| `end_date` | string | No | Fecha de apertura, fin de rango. Formato YYYY-MM-DD. |
| `final_period_start_date` | string | No | Fecha de cierre, inicio de rango. Formato YYYY-MM-DD. |
| `final_period_end_date` | string | No | Fecha de cierre, fin de rango. Formato YYYY-MM-DD. |
| `platform` | string | No | Slug de plataforma. |
| `office` | - | No | Filtro de oficina/organismo (int o string). |
| `bdns` | integer | No | Código BDNS. Debe enviarse como número entero. |
| `min_budget` | number | No | Presupuesto de la ayuda, mínimo de rango. |
| `max_budget` | number | No | Presupuesto de la ayuda, máximo de rango. |
| `order` | string | No | Atributo por el que ordenar los resultados. Antepón un guion (-) para orden descendente. Valores admitidos: `order_score` (afinidad/relevancia con la búsqueda, valor por defecto), `total_amount` (presupuesto), `start_date`, `end_date`, `final_period_start_date` y `final_period_end_date` (fechas). Ejemplos: `-total_amount` (mayor presupuesto primero), `-end_date` (cierre más próximo primero). |
| `zip_code` | string | No | Código postal por el que filtrar. |
| `status_code` | integer | No | Filtro por estado de la convocatoria: `0` = pendiente (aún no abierta), `1` = abierta, `2` = cerrada. Valores: 0, 1, 2. |
| `minimis` | boolean | No | Filtra por régimen de minimis. |
| `min_total_amount` | number | No | Importe total de la convocatoria, mínimo de rango. |
| `max_total_amount` | number | No | Importe total de la convocatoria, máximo de rango. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/concessions/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'requestData={"communities":[7],"applicants_v2":[1]}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "count": 58,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 55,
      "slug": "adelante-inversion-2025",
      "formatted_title": "Adelante Inversión",
      "status_text": "Cerrada",
      "total_amount": 5000000,
      "concessions_count": 1266,
      "concessions_amount": 4980000
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Concesiones por CIF — `GET /api/v2/funds/concessions/beneficiaries-by-cif/`

**Autenticación:** Token de usuario


Devuelve el listado de subvenciones/concesiones recibidas por una empresa concreta, identificada por su NIF/CIF: una fila por cada convocatoria que esa empresa ha recibido (nif fijo, fund_id variable). Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación. Algunas empresas tienen decenas de concesiones asociadas: si necesitas el total, sigue pidiendo páginas siguientes con page hasta agotar los resultados (usa count).


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `requestData` | query | object | Sí | Ejemplo: {"nif": "B88445358"}. |
| `page` | query | integer | No | Página del listado. Es un parámetro de query independiente, no va dentro de requestData. |

Campos de `requestData`:

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `nif` | string | Sí | CIF/NIF del beneficiario. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/concessions/beneficiaries-by-cif/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'requestData={"nif":"B88445358"}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "count": 15,
  "next": null,
  "previous": null,
  "results": [
    {
      "fund_id": 777794,
      "fund_slug": "ayudas-del-centro-para-el-desarrollo-tecnologico-y-la-innovacion-epe-para-la-financiacion-de-proyectos-de-id2023",
      "fund_title": "Ayudas del Centro para el Desarrollo Tecnológico y la Innovación para la financiación de proyectos de I+D en el año 2023.",
      "beneficiary_cif": "B88445358",
      "beneficiary_name": "FANDIT POWER SL",
      "concession_date": "2023-11-30",
      "awarded_amount": 225206.65
    },
    {
      "fund_id": 833338,
      "fund_slug": "subvenciones-dirigidas-al-fomento-de-modernizacion-tecnologica-y-digitalizacion-orientados-a-pymes-2024",
      "fund_title": "Subvenciones dirigidas al fomento de modernización tecnológica y digitalización de las PYMEs, año 2024.",
      "beneficiary_cif": "B88445358",
      "beneficiary_name": "FANDIT POWER SL . .",
      "concession_date": "2024-12-10",
      "awarded_amount": 50000
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**


- `400`: requestData inválido o no se envió nif.



---


#### Concesiones por subvención — `GET /api/v2/funds/concessions/beneficiaries/`

**Autenticación:** Token de usuario


Devuelve el listado de beneficiarios de una convocatoria concreta, identificada por fund_id o fund_slug: una fila por cada empresa que ha recibido esa convocatoria (fund_id fijo, beneficiario variable). Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación. Algunas convocatorias tienen más de 1000 concesiones asociadas: si necesitas el total, sigue pidiendo páginas siguientes con page hasta agotar los resultados (usa count).


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `requestData` | query | object | Sí | Ejemplo: {"fund_id": 916430}. Admite fund_id o fund_slug (no combinar ambos). |
| `page` | query | integer | No | Página del listado. Es un parámetro de query independiente, no va dentro de requestData. |

Campos de `requestData`:

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `fund_id` | integer | No | Id de la convocatoria. No combinar con fund_slug. |
| `fund_slug` | string | No | Slug de la convocatoria. No combinar con fund_id. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/concessions/beneficiaries/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'requestData={"fund_id":916430}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "count": 130,
  "next": "https://api.fandit.es/api/v2/funds/concessions/beneficiaries/?page=2&requestData=%7B%22fund_id%22%3A916430%7D",
  "previous": null,
  "results": [
    {
      "fund_id": 916430,
      "fund_slug": "ayudas-destinadas-a-nuevos-proyectos-empresariales-de-empresas-innovadora-programa-neotec-2025",
      "fund_title": "Ayudas destinadas a nuevos proyectos empresariales de empresas innovadora. Programa NEOTEC 2025.",
      "beneficiary_cif": "B44562163",
      "beneficiary_name": "EXXN ENGINEERING AI AND TELECOM SL",
      "concession_date": "2025-12-22",
      "awarded_amount": 335000
    },
    {
      "fund_id": 916430,
      "fund_slug": "ayudas-destinadas-a-nuevos-proyectos-empresariales-de-empresas-innovadora-programa-neotec-2025",
      "fund_title": "Ayudas destinadas a nuevos proyectos empresariales de empresas innovadora. Programa NEOTEC 2025.",
      "beneficiary_cif": "B72486095",
      "beneficiary_name": "MUSE SCENE LAB SL",
      "concession_date": "2025-12-22",
      "awarded_amount": 335000
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**


- `400`: requestData inválido o no se envió fund_id ni fund_slug.



---


#### Evaluación de una subvención — `GET /api/v2/funds/fund-evaluation/{id}/`

**Autenticación:** Token de usuario


Devuelve los criterios de evaluación de la convocatoria (requisitos y si está sujeta a régimen de minimis). Consultar siempre que se pregunte cómo aumentar probabilidades de éxito, qué se valora o cómo preparar mejor una solicitud, aunque no se mencione explícitamente "evaluación". Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | integer | Sí | Id de la convocatoria. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/fund-evaluation/5891/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 5891,
  "requirements": "Se evaluará el cumplimiento de los requisitos de admisibilidad y la disponibilidad de crédito presupuestario en el momento de la solicitud, por estricto orden de presentación.",
  "minimis": false
}
```


**Notas de errores específicas de este endpoint:**


- `404`: Convocatoria inexistente.



---


#### Normativa de una subvención — `GET /api/v2/funds/fund-normative/{id}/`

**Autenticación:** Token de usuario


Devuelve el listado de normativa disponible: enlaces, PDFs (cada uno con su URL directa y metadatos de clasificación), si es competitiva/no competitiva/minimis, BDNS. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | integer | Sí | Id de la convocatoria. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/fund-normative/5891/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "office": "Red.es",
  "department": "Ministerio para la Transformación Digital y de la Función Pública",
  "has_regulation_url": true,
  "pdf_files": [
    {
      "id": 5023,
      "title": "Bases reguladoras Kit Digital Segmento III",
      "filename": "bases_reguladoras_kit_digital_iii.pdf",
      "primary": true,
      "document_type": "Bases reguladoras",
      "coincidence_category": "Múltiple",
      "coincidence_percent": 0.95,
      "source": "https://s3.eu-west-1.amazonaws.com/media.fandit.es/files/bases_reguladoras_kit_digital_iii.pdf"
    },
    {
      "id": 5024,
      "title": "Modificación de las bases reguladoras",
      "filename": "modificacion_bases_kit_digital_iii.pdf",
      "primary": false,
      "document_type": "Modificaciones",
      "coincidence_category": "Múltiple",
      "coincidence_percent": 0.85,
      "source": "https://s3.eu-west-1.amazonaws.com/media.fandit.es/files/modificacion_bases_kit_digital_iii.pdf"
    }
  ],
  "has_link": true,
  "bdns_list": [
    "639942",
    "651203"
  ],
  "competitiva": false,
  "no_competitiva": true,
  "minimis": false
}
```


**Notas de errores específicas de este endpoint:**


- `404`: Convocatoria inexistente.



---


#### Subvenciones relacionadas — `GET /api/v2/funds/fund-related/{id}/`

**Autenticación:** Token de usuario


Devuelve el historial de ediciones anteriores de la misma ayuda, sin paginar. Cada elemento incluye el campo previous con el id de la edición inmediatamente anterior, lo que permite seguir recorriendo el historial hacia atrás si se necesita. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | integer | Sí | Id de la convocatoria. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/fund-related/916430/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
[
  {
    "register_date": "2025-05-02",
    "slug": "ayudas-destinadas-a-nuevos-proyectos-empresariales-de-empresas-innovadora-programa-neotec-2025",
    "cleaned_title": "Ayudas destinadas a nuevos proyectos empresariales de empresas innovadora. Programa NEOTEC 2025.",
    "status_text": "Apertura el 12/05/2025 y cierre el 12/06/2025",
    "total_amount": 40000000.0,
    "fund_scope": "Estatal",
    "concessions_count": 130,
    "concessions_amount": 39999999.99999999,
    "entity": "ESTADO",
    "department": "MINISTERIO DE CIENCIA, INNOVACIÓN Y UNIVERSIDADES",
    "office": "CENTRO PARA EL DESARROLLO TECNOLÓGICO Y LA INNOVACIÓN (CDTI)",
    "start_date": "2025-05-12",
    "previous": 916430
  },
  {
    "register_date": "2024-04-04",
    "slug": "ayudas-para-startups-tecnologicas-innovadoras-del-programa-neotec-ano-2024",
    "cleaned_title": "Ayudas destinadas a nuevos proyectos empresariales de empresas innovadora. Programa NEOTEC 2024.",
    "status_text": "Apertura el 10/04/2024 y cierre el 10/05/2024",
    "total_amount": 20000000.0,
    "fund_scope": "Estatal",
    "concessions_count": 64,
    "concessions_amount": 20000000.0,
    "entity": "ESTADO",
    "department": "MINISTERIO DE CIENCIA, INNOVACIÓN Y UNIVERSIDADES",
    "office": "CENTRO PARA EL DESARROLLO TECNOLÓGICO Y LA INNOVACIÓN (CDTI)",
    "start_date": "2024-04-10",
    "previous": 838550
  }
]
```


**Notas de errores específicas de este endpoint:**




---


#### Documentos requeridos de una subvención — `GET /api/v2/funds/fund-required-documents/{id}/`

**Autenticación:** Token de usuario


Devuelve la documentación requerida para solicitar la convocatoria. Combinar con fund-evaluation cuando se pregunte por estrategia de éxito: sin la documentación correcta ni siquiera se evalúa la solicitud. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `id` | path | integer | Sí | Id de la convocatoria. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/funds/fund-required-documents/5891/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "required_documents": "DNI o NIE del solicitante, certificado de estar al corriente de pagos con la Agencia Tributaria y la Seguridad Social, y formulario de solicitud cumplimentado.",
  "template_document": [
    {
      "id": 7745,
      "title": "Formulario de solicitud",
      "filename": "formulario_solicitud_kit_digital.pdf",
      "primary": true
    },
    {
      "id": 7746,
      "title": "Modelo de declaración responsable",
      "filename": "declaracion_responsable.pdf",
      "primary": false
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**


- `404`: Convocatoria inexistente.



---


#### Simulador por CIF — `POST /api/v2/funds/opportunities-by-cif/`

**Autenticación:** Token de usuario


Calcula hasta 20 oportunidades de subvención a partir del NIF/CIF y CNAE de una empresa: infiere el tipo de solicitante según el NIF y cruza con las acciones asociadas al CNAE. Para perfil genérico sin CIF usar el simulador por perfil. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `province` | integer | Sí | Provincia de la empresa. |
| `cnae` | string | Sí | Código CNAE 2009 de la empresa. |
| `nif` | string | Sí | CIF/NIF de la empresa. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/funds/opportunities-by-cif/' \
  --data '{ "province": 42, "cnae": "0111", "nif": "B88888888" }' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
[
  {
    "id": 1028474,
    "slug": "subvencion-programa-de-empleo-y-formacion-2026",
    "formatted_title": "Subvención Programa de empleo y formación",
    "status_text": "Apertura el 25/07/2026 y cierre el 11/09/2026",
    "total_amount": 108106379.11,
    "scope": "Andalucía"
  },
  {
    "id": 998414,
    "slug": "convocatoria-de-ayudas-para-solicitantes-que-no-son-grupos-de-desarrollo-rural-2026",
    "formatted_title": "Convocatoria de ayudas para solicitantes que no son Grupos de Desarrollo Rural 2026.  (Intervención 7119.2)",
    "status_text": "Apertura el 23/04/2026 y cierre el 31/12/2028",
    "total_amount": 97589590.75,
    "scope": "Andalucía"
  }
]
```


**Notas de errores específicas de este endpoint:**


- `400`: Faltan campos o el CNAE no es válido.



---


#### Simulador por perfil — `POST /api/v2/funds/opportunities-by-profile/`

**Autenticación:** Token de usuario


Igual que el simulador por CIF, pero recibiendo el perfil directamente en vez de derivarlo del NIF. Requiere token de usuario y créditos disponibles en la cuenta: devuelve 403 si el usuario no tiene créditos suficientes, distinto de un fallo de autenticación.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `province` | - | Sí | Provincia (int) o array de provincias. |
| `applicants_v2` | array[integer] | Sí | IDs de tipo de solicitante. No puede ir vacío. |
| `actions_v2` | array[integer] | Sí | IDs de acción/línea de ayuda. No puede ir vacío. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/funds/opportunities-by-profile/' \
  --header 'Authorization: Token TU_API_KEY' \
  --data '{"province": 42, "applicants_v2": [1,3,4,5,7,2,8,6], "actions_v2": [67,30,41,32,38,27,46,47,28,39,31,68,49,42,54,50,57,69,44,33,51,52,61,34,62,40,63,58,71,55,35,36,45,37,66,56,48,53,70,59,43,64,60,65,29]}'
```


**Ejemplo de respuesta:**


```json
[
  {
    "id": 1028474,
    "slug": "subvencion-programa-de-empleo-y-formacion-2026",
    "formatted_title": "Subvención Programa de empleo y formación",
    "status_text": "Apertura el 25/07/2026 y cierre el 11/09/2026",
    "total_amount": 108106379.11,
    "scope": "Andalucía"
  },
  {
    "id": 998414,
    "slug": "convocatoria-de-ayudas-para-solicitantes-que-no-son-grupos-de-desarrollo-rural-2026",
    "formatted_title": "Convocatoria de ayudas para solicitantes que no son Grupos de Desarrollo Rural 2026.  (Intervención 7119.2)",
    "status_text": "Apertura el 23/04/2026 y cierre el 31/12/2028",
    "total_amount": 97589590.75,
    "scope": "Andalucía"
  },
  {
    "id": 1033743,
    "slug": "ayudas-del-plan-de-emergencias-ante-el-riesgo-de-inundaciones-en-andalucia-peri-a-entidades-locales-especialmente-afectadas-por-fenomenos-naturales-adversos",
    "formatted_title": "Ayudas del Plan de Emergencias ante el Riesgo de Inundaciones en Andalucia (PERI) a entidades locales especialmente afectadas por fenómenos naturales adversos.",
    "status_text": "Abierta hasta agotar fondos",
    "total_amount": 35000000,
    "scope": "Andalucía"
  }
]
```


**Notas de errores específicas de este endpoint:**


- `400`: Faltan campos obligatorios o vienen vacíos.



---


#### Dashboard de subvenciones agregadas — `GET /api/v2/crm/fund-dashboard-data/`

**Autenticación:** Token de usuario o token de experto


Petición para obtener estadísticas agregadas de convocatorias activas (conteo e importe total) agrupadas por distintas dimensiones (comunidades, provincias, actividades, tipos de solicitante, tipos de convocatoria, acciones, orígenes y ámbito territorial) y por ventana temporal (día, semana, mes), dentro de un rango de fechas de alta de la convocatoria. Pensado para alimentar paneles/dashboards internos, no para el buscador de convocatorias en sí (para eso usa `GET /funds/`).


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `requestData` | query | object | Sí | Objeto de filtros serializado como JSON. Debe incluir como mínimo `start_date` y `end_date`. |

Campos de `requestData`:

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `start_date` | string | Sí | Fecha de alta de la convocatoria, inicio de rango. Formato YYYY-MM-DD. |
| `end_date` | string | Sí | Fecha de alta de la convocatoria, fin de rango. Formato YYYY-MM-DD. |
| `communities` | array[integer] | No | IDs de comunidad autónoma. |
| `provinces` | array[integer] | No | IDs de provincia. |
| `applicants_v2` | array[integer] | No | IDs de tipo de solicitante (mismo espacio de ids que `applicants` en /data-filters/). |
| `activities_v2` | array[integer] | No | IDs de actividad/sector. |
| `actions_v2` | array[integer] | No | IDs de acción/línea de ayuda. |
| `types` | array[integer] | No | IDs de tipo de convocatoria. |
| `region_types` | array[integer] | No | IDs de ámbito territorial. |
| `origins` | array[integer] | No | IDs de origen de fondos. |
| `order_by` | string | No | Campo de ordenación de los resultados agrupados (por defecto `count`). |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/crm/fund-dashboard-data/' \
  --get \
  --data-urlencode 'requestData={"start_date":"2026-01-01","end_date":"2026-06-30","communities":[13]}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "results": {
    "communities": {
      "months": [
        {
          "label": "2026-01-01 - 2026-01-31",
          "count": 42,
          "total": 15000000
        }
      ],
      "weeks": [],
      "days": [],
      "globals": [
        {
          "communities": 13,
          "name": "Comunidad de Madrid",
          "count": 42,
          "total": 15000000
        }
      ]
    },
    "provinces": { "months": [], "weeks": [], "days": [], "globals": [] },
    "activities_v2": { "months": [], "weeks": [], "days": [], "globals": [] },
    "applicant_types_v2": { "months": [], "weeks": [], "days": [], "globals": [] },
    "types": { "months": [], "weeks": [], "days": [], "globals": [] },
    "actions_v2": { "months": [], "weeks": [], "days": [], "globals": [] },
    "origins": { "months": [], "weeks": [], "days": [], "globals": [] },
    "total": 42,
    "total_amount": {
      "total_amount": 15000000
    },
    "values": {
      "days": [],
      "weeks": [],
      "months": []
    },
    "region_types": [
      {
        "region_type": 1,
        "count": 42,
        "total": 15000000,
        "name": "Nacional"
      }
    ]
  }
}
```

La respuesta no está paginada (se ignora `pagination_class` a nivel de implementación) y desglosa cada dimensión (`communities`, `provinces`, `activities_v2`, `applicant_types_v2`, `types`, `actions_v2`, `origins`) con la misma forma: `months`/`weeks`/`days` (series temporales) y `globals` (totales por valor de la dimensión). Los campos `total`, `total_amount`, `values` y `region_types` son agregados globales del conjunto completo de resultados filtrado.


**Notas de errores específicas de este endpoint:**


- `400`: Falta `start_date` o `end_date`, o el rango de fechas no es válido.



---


### Expedientes

#### Gestores disponibles — `GET /api/v2/experts/available-experts/`

**Autenticación:** Token de experto


Petición para obtener todos los gestores con los que se puede compartir o asignar expedientes y clientes.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/experts/available-experts/' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "count": 3,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 12,
      "email": "javier.ortega@example.com",
      "first_name": "Javier",
      "last_name": "Ortega"
    },
    {
      "id": 34,
      "email": "marta.sanchez@example.com",
      "first_name": "Marta",
      "last_name": "Sánchez"
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Listado de expedientes — `GET /api/v2/forms/`

**Autenticación:** Token de experto


Petición paginada para obtener todos los expedientes creados.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `page` | query | number | Sí | Página del listado a visualizar. |
| `page_size` | query | number | Sí | Tamaño de la paginación. |
| `isActive` | query | number | No | Mostrar solo los expedientes activos o no. |
| `general_text` | query | string | No | Texto a buscar en todos los campos del expediente. |
| `guest_email` | query | string | No | Correo electrónico del solicitante. |
| `guest_name` | query | string | No | Cliente o razón social. |
| `client_name` | query | string | No | contacto del expediente. |
| `status` | query | string | No | Estado del expediente. |
| `sub_status` | query | string | No | Subestado del expediente. |
| `start_date` | query | string | No | Fecha de creación (inicio de rango). |
| `end_date` | query | string | No | Fecha de creación (fin de rango). |
| `expert_name` | query | string | No | Nombre del gestor. |
| `fund_title` | query | string | No | Título del expediente. |
| `reference` | query | string | No | Referencia del expediente. |
| `concession_date_start` | query | string | No | Fecha de concesión (inicio de rango). |
| `concession_date_end` | query | string | No | Fecha de concesión (fin de rango). |
| `presentation_date_start` | query | string | No | Fecha de presentación (inicio de rango). |
| `presentation_date_end` | query | string | No | Fecha de presentación (fin de rango). |
| `requested_amount_min` | query | string | No | Monto solicitado (inicio de rango). |
| `requested_amount_max` | query | string | No | Monto solicitado (fin de rango). |
| `awarded_amount_min` | query | string | No | Monto concedido (inicio de rango). |
| `awarded_amount_max` | query | string | No | Monto concedido (fin de rango). |
| `my_workflows` | query | boolean | No | Mostrar solo los expediente asignados al experto logueado. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/forms/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'page_size=10' \
  --data-urlencode 'isActive=1' \
  --data-urlencode 'requestData={"general_text":"kit digital"}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "count": 128,
  "next": "https://api.fandit.es/api/v2/forms/?page=2&page_size=10",
  "previous": null,
  "results": [
    {
      "id": 4521,
      "identifier": "EXP-2026-04521",
      "uuid": "a1b2c3d4-5e6f-4a1b-8c9d-0e1f2a3b4c5d",
      "title": "Ayuda Kit Digital - Segmento III",
      "client_data": {
        "id": 982,
        "business_name": "Innovaciones Digitales SL",
        "email": "contacto@example.com",
        "nif": "B12345678"
      },
      "expert_data": {
        "id": 34,
        "name": "Marta Sánchez",
        "email": "marta.sanchez@example.com"
      },
      "company": "Innovaciones Digitales SL",
      "phone": "600000002",
      "platform": "fandit",
      "reference": "REF-2026-0088",
      "additional_data": "Solicitud vinculada a la convocatoria Kit Digital 2026",
      "status_group": {
        "id": 2,
        "name": "En trámite"
      },
      "current_status": {
        "id": 5,
        "name": "Pendiente de documentación"
      },
      "requested_amount": 12000,
      "awarded_amount": null,
      "presentation_date": "2026-03-14",
      "concession_date": null,
      "shared_client": true
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Crear expediente — `POST /api/v2/forms/`

**Autenticación:** Token de experto


Petición crear un nuevo expediente.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `template_id` | number | No | Id de plantilla de expediente. |
| `title` | string | Sí | Título del nuevo expediente. |
| `shared_client` | boolean | No | Compartir los datos del solicitante. |
| `expert_id` | number | No | Id del gestor asignado. |
| `client` | number | Sí | Id del cliente asignado. |
| `applicant_additional_data` | string | No | Nº Expediente (información adicional). |
| `applicant_reference` | string | No | Referencia. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/forms/' \
  --data '{"template_id":58,"title":"Ayuda Kit Digital - Segmento III","shared_client":true,"expert_id":34,"client":982,"applicant_additional_data":"Solicitud vinculada a la convocatoria Kit Digital 2026","applicant_reference":"REF-2026-0088"}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "workflow_id": 4837,
  "exist_applicant": true,
  "uuid": "f3a1b2c4-6d7e-4f8a-9b0c-1d2e3f4a5b6c"
}
```


**Notas de errores específicas de este endpoint:**




---


#### Asignar expedientes a un gestor — `POST /api/v2/forms/expert-assign/`

**Autenticación:** Token de experto


Petición para asignarle un mismo gestor a expedientes de forma masiva.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `workflows` | array[integer] | Sí | Array de ids de expedientes a actualizar. |
| `expert_id` | number | Sí | Id del experto a asignar. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/forms/expert-assign/' \
  --data '{"workflows":[4521,4498,4390],"expert_id":34}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "response": "Se han asignado 3 expedientes al gestor seleccionado."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Importar expedientes — `POST /api/v2/forms/list-import/`

**Autenticación:** Token de experto


Petición para importar expedientes de forma masiva a partir de un excel.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `file` | string | Sí | Archivo con expedientes a importar. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/forms/list-import/' \
  --File 'Remove-Content-Type=True' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "response": "Se han importado 12 expedientes correctamente."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Estados y subestados de expedientes — `GET /api/v2/forms/status-groups-list/`

**Autenticación:** Token de experto


Petición paginada para obtener todos los posibles estados y subestados que puede tener un expediente, y poder seleccionarlos al momento de crear uno.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/forms/status-groups-list/' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "count": 6,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "name": "En trámite",
      "status": [
        {
          "id": 1,
          "initial_state": true,
          "name": "Pendiente de documentación",
          "template": 1
        },
        {
          "id": 2,
          "initial_state": false,
          "name": "En revisión",
          "template": 1
        }
      ]
    },
    {
      "id": 2,
      "name": "Concedido",
      "status": [
        {
          "id": 3,
          "initial_state": false,
          "name": "Concedido - pendiente de justificación",
          "template": 1
        },
        {
          "id": 4,
          "initial_state": false,
          "name": "Justificado",
          "template": 1
        }
      ]
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Actualizar estados y subestados de expedientes — `POST /api/v2/forms/update-status/`

**Autenticación:** Token de experto


Petición para actualizar el estado de expedientes de forma masiva.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `workflows` | array[integer] | Sí | Array de ids de expedientes a actualizar. |
| `new_status_group` | number | Sí | Estado del expediente. |
| `new_status` | number | Sí | Subestado del expediente. |
| `observation` | string | No | Nota de observación. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/forms/update-status/' \
  --data '{"workflows":[4521,4498,4390],"new_status_group":2,"new_status":9,"observation":"Actualización masiva de estado tras revisión documental."}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "response": "Se han actualizado 3 expedientes correctamente."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Eliminar, archivar o restaurar expedientes — `POST /api/v2/forms/update/`

**Autenticación:** Token de experto


Petición para eliminar, archivar o restaurar expedientes de forma masiva.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `workflows` | array[integer] | Sí | Array de ids de expedientes a actualizar. |
| `action` | string | Sí | Acción a llevar a cabo (Eliminar, archivar, desarchivar). |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/forms/update/' \
  --data '{"workflows":[4521,4498,4390],"action":"archive"}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "response": "Se han archivado 3 expedientes correctamente."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Detalle de un expediente — `GET /api/v2/forms/{workflow_id}/`

**Autenticación:** Token de experto


Petición para obtener toda la información de un expediente específico.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `workflow_id` | path | string | Sí | Id del expediente a buscar. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/forms/4521/' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "id": 4521,
  "identifier": "EXP-2026-04521",
  "uuid": "a1b2c3d4-5e6f-4a1b-8c9d-0e1f2a3b4c5d",
  "title": "Ayuda Kit Digital - Segmento III",
  "client_data": {
    "id": 982,
    "business_name": "Innovaciones Digitales SL",
    "email": "contacto@example.com",
    "nif": "B12345678"
  },
  "expert_data": {
    "id": 34,
    "name": "Marta Sánchez",
    "email": "marta.sanchez@example.com"
  },
  "company": "Innovaciones Digitales SL",
  "phone": "600000002",
  "platform": "fandit",
  "reference": "REF-2026-0088",
  "additional_data": "Solicitud vinculada a la convocatoria Kit Digital 2026",
  "status_group": {
    "id": 2,
    "name": "En trámite"
  },
  "current_status": {
    "id": 5,
    "name": "Pendiente de documentación"
  },
  "requested_amount": 12000,
  "awarded_amount": null,
  "presentation_date": "2026-03-14",
  "concession_date": null,
  "shared_client": true
}
```


**Notas de errores específicas de este endpoint:**




---


#### Plantillas de expedientes — `GET /api/v2/templates/`

**Autenticación:** Token de experto


Petición paginada para obtener todas las plantillas necesarias para la creación de un expediente.


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/templates/' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "count": 14,
  "next": "https://api.fandit.es/api/v2/templates/?page=2",
  "previous": null,
  "results": [
    {
      "copied": false,
      "created_at": "2025-11-04T09:32:11Z",
      "custom_forms": {
        "mine": 3,
        "others": 8
      },
      "expert_data": {
        "email": "marta.sanchez@example.com",
        "id": 34,
        "name": "Marta Sánchez"
      },
      "experts": [
        12,
        34,
        56
      ],
      "fund_data": {
        "id": 210,
        "title": "Kit Digital - Segmento III"
      },
      "id": 58,
      "platform": "fandit",
      "public": true,
      "title": "Plantilla Kit Digital",
      "updated_at": "2026-02-18T14:05:44Z"
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


### Clientes

#### Listado de clientes — `GET /api/v2/clients/`

**Autenticación:** Token de experto


Petición paginada para obtener todos los clientes que correspondan según el permiso del gestor.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `page` | query | number | Sí | Página del listado a visualizar. |
| `page_size` | query | number | Sí | Tamaño de la paginación (Obligatorio, máximo 200). |
| `order` | query | string | No | Nombre de atributo por el que ordenar los resultados. |
| `general_text` | query | string | No | Texto a buscar en todos los campos del cliente. |
| `email__contains` | query | string | No | Correo electrónico del cliente. |
| `nif__icontains` | query | string | No | NIF/CIF del cliente. |
| `reference__icontains` | query | string | No | Referencia del cliente. |
| `business_name__icontains` | query | string | No | Nombre o razón social. |
| `contacts__icontains` | query | string | No | Contacto del cliente. |
| `simulator_check` | query | string | No | Cliente con perfil de solicitante completo. |
| `status__in` | query | string | No | Estado del usuario. |
| `expert__in` | query | array[integer] | No | Mostrar solo mis clientes asignados. |
| `provinces_to_work__id__in` | query | array[integer] | No | Provincias de interés del cliente. |
| `applicant_types_v2__id__in` | query | array[integer] | No | Tipo de solicitante (mismo espacio de ids que `applicants` en /data-filters/). |
| `province_id__in` | query | array[integer] | No | Provincia de la sede. |
| `cnaes__id__in` | query | array[integer] | No | CNAE(s) del cliente. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/clients/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'page_size=10' \
  --data-urlencode 'filters={"business_name__icontains":"innovatech","status__in":"1"}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "count": 128,
  "next": "https://api.fandit.es/api/v2/clients/?page=2&page_size=10",
  "previous": null,
  "results": [
    {
      "id": 3021,
      "expert_data": {
        "id": 45,
        "name": "María Fernández",
        "email": "maria.fernandez@example.com"
      },
      "experts": [
        45,
        52
      ],
      "public": false,
      "status": 1,
      "platform_data": {
        "id": 2,
        "name": "FANDIT"
      },
      "nif": "B12345678",
      "business_name": "Innovatech Soluciones SL",
      "email": "contacto@example.com",
      "phone": null,
      "reference": null,
      "notes": null,
      "province": [
        28
      ],
      "address": "Calle Alcalá 120, 3ºB",
      "postal_code": "28009",
      "location": "Madrid",
      "legal_representative_name": "Carlos Ruiz Gómez",
      "legal_representative_nif": "12345678Z",
      "legal_representation_type": "Administrador único",
      "contacts": "Carlos Ruiz Gómez - 600000001",
      "cnaes": [
        6201,
        6202
      ],
      "provinces_to_work": [
        28,
        8
      ],
      "applicant_types": [
        1,
        3
      ],
      "action_items": [
        1,
        2
      ],
      "employees_quantity": 18,
      "constitution_date": "2015-03-12",
      "last_year_billing": 850000,
      "investment_budget": 120000,
      "project_description": "Digitalización de procesos internos y desarrollo de nueva plataforma de gestión."
    },
    {
      "id": 3045,
      "expert_data": {
        "id": 52,
        "name": "Javier Ortega",
        "email": "javier.ortega@example.com"
      },
      "experts": [
        52
      ],
      "public": true,
      "status": 1,
      "platform_data": {
        "id": 2,
        "name": "FANDIT"
      },
      "nif": "44556677Q",
      "business_name": "Panadería Hermanos Soler",
      "email": "info@example.com",
      "phone": null,
      "reference": "REF-2024-0198",
      "notes": "Cliente interesado en ayudas de digitalización",
      "province": [
        8
      ],
      "address": "Avinguda Diagonal 455",
      "postal_code": "08036",
      "location": "Barcelona",
      "legal_representative_name": "Marta Soler Puig",
      "legal_representative_nif": "87654321X",
      "legal_representation_type": "Autónomo",
      "contacts": "Marta Soler Puig - 600000005",
      "cnaes": [
        1071
      ],
      "provinces_to_work": [
        8
      ],
      "applicant_types": [
        2
      ],
      "action_items": [
        3
      ],
      "employees_quantity": 6,
      "constitution_date": "2019-07-01",
      "last_year_billing": 210000,
      "investment_budget": 35000,
      "project_description": "Ampliación de obrador y compra de maquinaria."
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Crear cliente — `POST /api/v2/clients/`

**Autenticación:** Token de experto


Petición crear un nuevo cliente.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `nif` | string | Sí | NIF/CIF del cliente. |
| `business_name` | string | No | Nombre o razón social del cliente. |
| `email` | string | No | Correo electrónico del cliente. |
| `phone` | string | No | Teléfono del cliente. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/clients/' \
  --data '{"nif":"B98765432","business_name":"Talleres Mecánicos Rivas SL","email":"admin@example.com","phone":"600000006"}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "id": 3102,
  "expert_data": {
    "id": 45,
    "name": "María Fernández",
    "email": "maria.fernandez@example.com"
  },
  "experts": [
    45
  ],
  "public": false,
  "status": 1,
  "platform_data": {
    "id": 2,
    "name": "FANDIT"
  },
  "nif": "B98765432",
  "business_name": "Talleres Mecánicos Rivas SL",
  "email": "admin@example.com",
  "phone": "600000006",
  "reference": null,
  "notes": null,
  "province": [],
  "address": "",
  "postal_code": "",
  "location": "",
  "legal_representative_name": "",
  "legal_representative_nif": "",
  "legal_representation_type": "",
  "contacts": "",
  "cnaes": [],
  "provinces_to_work": [],
  "applicant_types": [],
  "action_items": [],
  "employees_quantity": null,
  "constitution_date": null,
  "last_year_billing": null,
  "investment_budget": null,
  "project_description": ""
}
```


**Notas de errores específicas de este endpoint:**




---


#### Compartir clientes — `POST /api/v2/clients/share/`

**Autenticación:** Token de experto


Petición para compartir clientes con otros gestores.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `client_ids` | array[integer] | Sí | Array de ids de clientes a compartir (Obligatorio, formato: 1, 2, 3). |
| `experts` | array[integer] | Sí | Array de ids de experts con los que compartir (Obligatorio, formato: 1, 2, 3). |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/clients/share/' \
  --data '{"client_ids":[3021,3045,3102],"experts":[45,52]}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "response": "Se han compartido 3 cliente(s) con 2 experto(s) correctamente."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Eliminar cliente — `DELETE /api/v2/clients/{client_id}/`

**Autenticación:** Token de experto


Petición un cliente en específico.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `client_id` | path | string | Sí | Id del cliente. |


**Ejemplo de petición:**


```bash
curl --request DELETE \
  --url 'https://api.fandit.es/api/v2/clients/3021/' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "Response": "Cliente eliminado correctamente."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Detalle del cliente — `GET /api/v2/clients/{client_id}/`

**Autenticación:** Token de experto


Petición para obtener toda la información de una cliente específico.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `client_id` | path | string | Sí | client_id del cliente a buscar. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/clients/3021/' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "id": 3021,
  "expert_data": {
    "id": 45,
    "name": "María Fernández",
    "email": "maria.fernandez@example.com"
  },
  "experts": [
    45,
    52
  ],
  "public": false,
  "status": 1,
  "platform_data": {
    "id": 2,
    "name": "FANDIT"
  },
  "nif": "B12345678",
  "business_name": "Innovatech Soluciones SL",
  "email": "contacto@example.com",
  "phone": null,
  "reference": null,
  "notes": null,
  "province": [
    28
  ],
  "address": "Calle Alcalá 120, 3ºB",
  "postal_code": "28009",
  "location": "Madrid",
  "legal_representative_name": "Carlos Ruiz Gómez",
  "legal_representative_nif": "12345678Z",
  "legal_representation_type": "Administrador único",
  "contacts": "Carlos Ruiz Gómez - 600000001",
  "cnaes": [
    6201,
    6202
  ],
  "provinces_to_work": [
    28,
    8
  ],
  "applicant_types": [
    1,
    3
  ],
  "action_items": [
    1,
    2
  ],
  "employees_quantity": 18,
  "constitution_date": "2015-03-12",
  "last_year_billing": 850000,
  "investment_budget": 120000,
  "project_description": "Digitalización de procesos internos y desarrollo de nueva plataforma de gestión."
}
```


**Notas de errores específicas de este endpoint:**




---


#### Actualizar cliente — `PATCH /api/v2/clients/{client_id}/`

**Autenticación:** Token de experto


Petición para actualizar los datos de un cliente específico. `PUT` también está soportado en esta misma URL y se comporta de forma idéntica a `PATCH` (actualización parcial en ambos casos); se documenta solo `PATCH` porque es el método recomendado.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `client_id` | path | string | Sí | Identificador del cliente a actualizar. |


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `nif` | string | No | NIF/CIF del cliente. |
| `business_name` | string | No | Nombre o razón social del cliente. |
| `email` | string | No | Correo electrónico del cliente. |
| `phone` | string | No | Teléfono del cliente. |
| `public` | boolean | No | Cliente disponible para compartir con otros gestores. |
| `status` | string | No | Estado del usuario. |
| `address` | string | No | Dirección del cliente. |
| `postal_code` | string | No | Codigo postal del cliente. |
| `location` | string | No | Localización del cliente. |
| `legal_representative_name` | string | No | Nombre de representante legal. |
| `legal_representative_nif` | string | No | NIF/CIF de representante legal. |
| `legal_representation_type` | string | No | Tipo de representación legal. |
| `province` | number | No | Provincia de la sede. |
| `cnaes` | array[integer] | No | CNAE(s) del cliente. |
| `provinces_to_work` | array[integer] | No | Provincias de interés del cliente. |
| `applicant_types` | array[integer] | No | Tipo de solicitante. |
| `action_items` | array[integer] | No | Acciones a llevar a cabo. |
| `employees_quantity` | number | No | Cantidad de empleados. |
| `constitution_date` | string | No | Fecha de constitución. |
| `last_year_billing` | number | No | Facturación anual. |
| `investment_budget` | number | No | Inversión prevista. |
| `project_description` | string | No | Descripción del proyecto. |


**Ejemplo de petición:**


```bash
curl --request PATCH \
  --url 'https://api.fandit.es/api/v2/clients/3021/' \
  --data '{"business_name":"Innovatech Soluciones SL","phone":"600000007","public":true,"employees_quantity":22,"last_year_billing":920000,"investment_budget":150000}' \
  --header 'Authorization: ExpertToken TU_EXPERT_TOKEN'
```


**Ejemplo de respuesta:**


```json
{
  "id": 3021,
  "expert_data": {
    "id": 45,
    "name": "María Fernández",
    "email": "maria.fernandez@example.com"
  },
  "experts": [
    45,
    52
  ],
  "public": true,
  "status": 1,
  "platform_data": {
    "id": 2,
    "name": "FANDIT"
  },
  "nif": "B12345678",
  "business_name": "Innovatech Soluciones SL",
  "email": "contacto@example.com",
  "phone": "600000007",
  "reference": null,
  "notes": null,
  "province": [
    28
  ],
  "address": "Calle Alcalá 120, 3ºB",
  "postal_code": "28009",
  "location": "Madrid",
  "legal_representative_name": "Carlos Ruiz Gómez",
  "legal_representative_nif": "12345678Z",
  "legal_representation_type": "Administrador único",
  "contacts": "Carlos Ruiz Gómez - 600000001",
  "cnaes": [
    6201,
    6202
  ],
  "provinces_to_work": [
    28,
    8
  ],
  "applicant_types": [
    1,
    3
  ],
  "action_items": [
    1,
    2
  ],
  "employees_quantity": 22,
  "constitution_date": "2015-03-12",
  "last_year_billing": 920000,
  "investment_budget": 150000,
  "project_description": "Digitalización de procesos internos y desarrollo de nueva plataforma de gestión."
}
```


**Notas de errores específicas de este endpoint:**




---


### Contactos

#### Listado de contactos — `GET /api/v2/summary-topics/`

**Autenticación:** Token de experto


Petición paginada para obtener todos los contactos registrados.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `page` | query | number | Sí | Página del listado a visualizar. |
| `page_size` | query | number | Sí | Tamaño de la paginación (Obligatorio, máximo 200). |
| `order` | query | string | No | Nombre de atributo por el que ordenar los resultados. |
| `general_text` | query | string | No | Texto a buscar en todas las columnas. |
| `email` | query | string | No | Correo electrónico a buscar. |
| `company` | query | string | No | Nombre o razón social. |
| `nif` | query | string | No | NIF/CIF. |
| `start_date` | query | string | No | Fecha de último contacto (inicio de rango). |
| `end_date` | query | string | No | Fecha de último contacto (fin de rango). |
| `unregistered_users` | query | number | No | Contactos con usuario registrado o no. |
| `provinces` | query | array[integer] | No | Provincia. |
| `applicants` | query | array[integer] | No | Tipo de solicitante. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/summary-topics/' \
  --get \
  --data-urlencode 'page=1' \
  --data-urlencode 'page_size=20' \
  --data-urlencode 'filters={"company":"innovatech"}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 3021,
      "email": "contacto@example.com",
      "name": "Ana López",
      "company": "Empresa Ejemplo SL",
      "cif": "B12345678",
      "contact_type": 1,
      "contact_type_label": "Solicitud web",
      "user": 10482,
      "contacts": 3,
      "last_contact_date": "2026-07-28",
      "created_at": "2026-06-15",
      "lead_status": 3,
      "lead_priority": 2,
      "status_label": "Interés",
      "priority_label": "Media",
      "investment_capital": 150000,
      "project_description": "Ampliación de planta de producción y digitalización de procesos",
      "applicants": [
        1
      ],
      "provinces": [
        28
      ],
      "action_items": [
        1,
        2
      ],
      "cnaes": [
        4321
      ]
    },
    {
      "id": 3045,
      "email": "info@example.com",
      "name": "Carlos Fernández",
      "company": "Innovatech Soluciones SL",
      "cif": "B87654321",
      "contact_type": 2,
      "contact_type_label": "Referido",
      "user": 10501,
      "contacts": 1,
      "last_contact_date": "2026-08-02",
      "created_at": "2026-08-01",
      "lead_status": 1,
      "lead_priority": 3,
      "status_label": "Nuevo",
      "priority_label": "Alta",
      "investment_capital": 320000,
      "project_description": "Implantación de sistema de energía solar fotovoltaica",
      "applicants": [
        2
      ],
      "provinces": [
        46
      ],
      "action_items": [
        3
      ],
      "cnaes": [
        4322
      ]
    }
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Crear contacto — `POST /api/v2/summary-topics/`

**Autenticación:** Token de experto


Petición crear un nuevo contacto.


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo electrónico del contacto. |
| `name` | string | No | Nombre. |
| `company` | string | No | Nombre o razón social. |
| `phone` | string | No | Teléfono. |
| `lead_status` | number | No | Estado del contacto ( Nuevo = 1, Toma de contacto = 2, Interés = 3, En evolución = 4, No me interesa = 5, Contratado = 6, Cerrado = 7). |
| `priority_status` | number | No | Prioridad del contacto (Baja = 1, Media = 2, Alta = 3). |
| `investment_capital` | number | No | Inversión prevista. |
| `project_description` | string | No | Descripción del proyecto. |
| `applicants` | array[integer] | No | Tipo de solicitante. |
| `provinces` | array[integer] | No | Provincia. |
| `action_items` | array[integer] | No | Acciones a llevar a cabo. |
| `cnaes` | array[integer] | No | CNAE(s) del contacto. |


**Ejemplo de petición:**


```bash
curl --request POST \
  --url 'https://api.fandit.es/api/v2/summary-topics/' \
  --data '{"email":"nuevo.contacto@example.com","name":"Marta Ruiz","company":"Consultora Ejemplo SL","phone":"600000002","lead_status":1,"priority_status":1,"investment_capital":80000,"project_description":"Proyecto de digitalización comercial","applicants":[1],"provinces":[28],"action_items":[1],"cnaes":[4321]}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 3102,
  "email": "nuevo.contacto@example.com",
  "name": "Marta Ruiz",
  "company": "Consultora Ejemplo SL",
  "cif": "B11223344",
  "contact_type": 1,
  "contact_type_label": "Solicitud web",
  "user": null,
  "contacts": 1,
  "last_contact_date": "2026-08-10",
  "created_at": "2026-08-10",
  "lead_status": 1,
  "lead_priority": 1,
  "status_label": "Nuevo",
  "priority_label": "Baja",
  "investment_capital": 80000,
  "project_description": "Proyecto de digitalización comercial",
  "applicants": [
    1
  ],
  "provinces": [
    28
  ],
  "action_items": [
    1
  ],
  "cnaes": [
    4321
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Detalle del contacto — `GET /api/v2/summary-topics/{contact_id}/`

**Autenticación:** Token de experto


Petición paginada para obtener todos los contactos registrados.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `contact_id` | path | string | Sí | id del contacto a buscar. |


**Ejemplo de petición:**


```bash
curl --request GET \
  --url 'https://api.fandit.es/api/v2/summary-topics/3021/' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 3021,
  "email": "contacto@example.com",
  "name": "Ana López",
  "company": "Empresa Ejemplo SL",
  "cif": "B12345678",
  "contact_type": 1,
  "contact_type_label": "Solicitud web",
  "user": 10482,
  "contacts": 3,
  "last_contact_date": "2026-07-28",
  "created_at": "2026-06-15",
  "lead_status": 3,
  "lead_priority": 2,
  "status_label": "Interés",
  "priority_label": "Media",
  "investment_capital": 150000,
  "project_description": "Ampliación de planta de producción y digitalización de procesos",
  "applicants": [
    1
  ],
  "provinces": [
    28
  ],
  "action_items": [
    1,
    2
  ],
  "cnaes": [
    4321
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


#### Actualizar contacto — `PATCH /api/v2/summary-topics/{contact_id}/`

**Autenticación:** Token de experto


Petición actualizar un contacto en específico.


**Parámetros:**


| Nombre | Ubicación | Tipo | Obligatorio | Descripción |
|---|---|---|---|---|
| `contact_id` | path | string | Sí | Identificador del contacto a actualizar. |


**Cuerpo de la petición (JSON):**


| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `name` | string | No | Nombre. |
| `company` | string | No | Nombre o razón social. |
| `phone` | string | No | Teléfono. |
| `lead_status` | number | No | Estado del contacto ( Nuevo = 1, Toma de contacto = 2, Interés = 3, En evolución = 4, No me interesa = 5, Contratado = 6, Cerrado = 7). |
| `priority_status` | number | No | Prioridad del contacto (Baja = 1, Media = 2, Alta = 3). |
| `investment_capital` | number | No | Inversión prevista. |
| `project_description` | string | No | Descripción del proyecto. |
| `applicants` | array[integer] | No | Tipo de solicitante. |
| `provinces` | array[integer] | No | Provincia. |
| `action_items` | array[integer] | No | Acciones a llevar a cabo. |
| `cnaes` | array[integer] | No | CNAE(s) del contacto. |


**Ejemplo de petición:**


```bash
curl --request PATCH \
  --url 'https://api.fandit.es/api/v2/summary-topics/3021/' \
  --data '{"lead_status":4,"priority_status":3,"investment_capital":175000,"action_items":[1,2,3]}' \
  --header 'Authorization: Token TU_API_KEY'
```


**Ejemplo de respuesta:**


```json
{
  "id": 3021,
  "email": "contacto@example.com",
  "name": "Ana López",
  "company": "Empresa Ejemplo SL",
  "cif": "B12345678",
  "contact_type": 1,
  "contact_type_label": "Solicitud web",
  "user": 10482,
  "contacts": 4,
  "last_contact_date": "2026-08-09",
  "created_at": "2026-06-15",
  "lead_status": 4,
  "lead_priority": 3,
  "status_label": "En evolución",
  "priority_label": "Alta",
  "investment_capital": 175000,
  "project_description": "Ampliación de planta de producción y digitalización de procesos",
  "applicants": [
    1
  ],
  "provinces": [
    28
  ],
  "action_items": [
    1,
    2,
    3
  ],
  "cnaes": [
    4321
  ]
}
```


**Notas de errores específicas de este endpoint:**




---


## Apéndice: qué endpoint usar según la necesidad

| Si se necesita... | Usar |
|---|---|
| Traducir un filtro en texto libre a un id numérico | `GET /data-filters/` (o dejar que `GET /funds/` lo infiera vía `search_by_vectorized_text`) |
| Explorar o filtrar el catálogo general de convocatorias abiertas | `GET /funds/` |
| Ver el detalle completo de una convocatoria activa concreta | `GET /fund-details/{identifier}/` (id o slug) |
| Ver convocatorias ya resueltas / con concesiones publicadas | `GET /funds/concessions/` |
| Ver qué ayudas ha recibido una empresa concreta (por CIF) | `GET /funds/concessions/beneficiaries-by-cif/` con `nif` |
| Ver quién recibió una convocatoria concreta | `GET /funds/concessions/beneficiaries/` con `fund_id`/`fund_slug` |
| Saber a qué puede optar una empresa con CIF/CNAE conocidos | `POST /funds/opportunities-by-cif/` |
| Saber a qué puede optar un perfil genérico sin CIF | `POST /funds/opportunities-by-profile/` |
| Ver la normativa/documentación legal de una convocatoria | `GET /funds/fund-normative/{id}/` |
| Saber cómo se evalúan las solicitudes | `GET /funds/fund-evaluation/{id}/` |
| Ver ediciones anteriores de la misma ayuda | `GET /funds/fund-related/{id}/` |
| Saber qué documentación hay que aportar | `GET /funds/fund-required-documents/{id}/` |
| Responder una pregunta abierta sobre una convocatoria concreta | `POST /funds/chatbot/` |
| Evaluar estrategia / probabilidad de éxito de una solicitud | Combinar `fund-evaluation` **+** `fund-required-documents`, no solo el detalle |
| Gestionar usuarios, clientes, contactos o expedientes | Bloques correspondientes con **token de experto** |

