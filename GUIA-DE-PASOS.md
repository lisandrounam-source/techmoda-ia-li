# Guía de pasos — cómo pegar los snippets de cada sesión en `template.yaml`

> Archivo pedido como *"Guía de pasos"*; lo nombré `GUIA-DE-PASOS.md` para seguir la convención del
> repo (`GUIA.md`, `GUIA-COMPLETA.md`) y que no tenga espacios en el path.

**Stack de trabajo:** `techmoda-ai-li` · **Región:** `us-east-1` · **Template:** [template.yaml](template.yaml)

**Estado hoy:** pegadas **S00 (base) + S01 + S02 + S03**. La próxima es **S04 (Translate)**.

---

## 0. Qué se arregló y por qué se duplicaban los pasos

`template.yaml` había quedado así después de pegar S01, S02 y S03:

| Problema | Qué provocaba |
|---|---|
| Recursos en la **columna 0** en vez de dentro de `Resources:` | `Invalid template property or properties [FrontendDistribution, RouterFunction, EnrichLabelsFunction, FrontendBucket, FrontendBucketPolicy]` → el changeset falla y **no despliega nada** |
| **Dos bloques `Outputs:`** | YAML se queda con el último → `ApiUrl`, `FrontendUrl`, etc. **desaparecen** del stack sin ningún error |
| Recursos en orden mezclado (S01 entre la tabla y el router) | imposible ver de un vistazo qué sesión ya está pegada → se repega y aparece `Duplicate found` |
| Los **bloques de comentarios del final** de cada snippet copiados también | se acumulaban 3 copias de ejemplos de `curl` (30 líneas de basura); es lo que hacía "duplicar los pasos" visualmente |
| `ModerateImageUrl` con 3 espacios y `AnalyzeSentimentUrl` con 5 | por suerte YAML lo tolera, pero rompe el patrón y confunde al pegar el siguiente |

Lo que hice:

1. Todos los recursos con **2 espacios** dentro de `Resources:`.
2. Un **solo** `Outputs:`.
3. Reordenado en dos regiones separadas por banderas: `BASE (S00)` primero, `SESIONES DE IA` después con S01 → S02 → S03 en orden y un encabezado `# ── SNN · servicio ──` por sesión.
4. Borrados los 30 renglones de comentarios sobrantes de los snippets.
5. Outputs normalizados a 2 espacios y **con `Description:`** cada uno.
6. **Dos anclas** (§1) que dicen exactamente dónde pegar.
7. Un **inventario** en la cabecera del archivo: qué sesiones ya están pegadas y cuál sigue.

**Nada de esto cambia un solo logical ID**, así que el próximo `sam deploy` no reemplaza ni recrea
ningún recurso: solo agrega las `Description` de los outputs (metadata).

Verificado:

```
$ sam validate --lint -t template.yaml
/workshop/capstone/template.yaml is a valid SAM Template

recursos: ProductsTable, RouterFunction, FrontendBucket, FrontendBucketPolicy,
          FrontendDistribution, EnrichLabelsFunction, ModerateImageFunction,
          AnalyzeSentimentFunction                                            (8, ninguno perdido)
outputs:  ApiUrl, ProductsTableName, Region, FrontendUrl, FrontendBucketName,
          EnrichLabelsUrl, ModerateImageUrl, AnalyzeSentimentUrl              (8 = los 8 del stack)
```

---

## 1. Las dos anclas

Cada snippet aporta **dos cosas** que van a **dos lugares distintos** del template. Buscalas con:

```bash
grep -n "ANCLA" template.yaml
```

```
245:  # ║  ANCLA 1 — PEGAR ACÁ EL RECURSO DEL PRÓXIMO SNIPPET (siguiente: S04)    ║
248:  # ║  va en el ANCLA 2, más abajo.                                          ║
286:  # ║  ANCLA 2 — PEGAR ACÁ EL OUTPUT DEL PRÓXIMO SNIPPET (siguiente: S04)     ║
```

- **ANCLA 1** (dentro de `Resources:`, al final) → el/los **recursos** del snippet.
- **ANCLA 2** (dentro de `Outputs:`, al final) → el/los **outputs** de Function URL.

Pegá **arriba** de la caja del ancla, no adentro ni debajo: así el ancla queda siempre al final de su
sección y sirve para la sesión siguiente.

### Anatomía de un `template-snippet.yaml`

Todos tienen la misma forma. Ejemplo real, S04:

```yaml
# ============================================================================     ← ①  NO copiar
# S4 · Amazon Translate  —  pegar en template.yaml
# ============================================================================
# Patrón: Function URL propia (sin API Gateway) + `Policies:` de mínimo privilegio
# ----------------------------------------------------------------------------

  TranslateCatalogFunction:                                                    ← ②  ESTO va a ANCLA 1
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: !Sub ${AWS::StackName}-TranslateCatalog
      ...

# ----------------------------------------------------------------------------     ← ③  NO copiar tal cual
# FUNCTION URL — Outputs:
#   TranslateCatalogUrl:                                                       ←     de acá sale el
#     Value: !GetAtt TranslateCatalogFunctionUrl.FunctionUrl                   ←     Output de ANCLA 2
#
# Invocación (id por path o body; target en el body):
#   curl -X POST "${URL%/}/products/<PRODUCT_ID>/translate" ...                ←     esto es documentación
# ----------------------------------------------------------------------------
```

| Parte | Qué es | Qué hacer |
|---|---|---|
| ① cabecera `#` | explicación de la sesión | **no copiar** |
| ② cuerpo indentado 2 espacios | el recurso SAM | copiar **tal cual** a ANCLA 1 |
| ③ pie `#` | el Output **comentado** + ejemplos de `curl` | **no copiar el bloque**; solo reescribir el Output en ANCLA 2 |

**Copiar el pie entero es el error que ensuciaba el template.** El `curl` de ejemplo no va en el
template: ya está en el `GUIA.md` de la sesión.

---

## 2. Receta — los 6 pasos, iguales para cualquier sesión

Con S04 como ejemplo. Todo desde `/workshop/capstone`.

### Paso 1 — Chequeo anti-duplicado (30 segundos que ahorran 10 minutos)

```bash
grep -nE '^ *(TranslateCatalogFunction|TranslateCatalogUrl):' template.yaml
```

**Qué hace:** busca el logical ID del recurso y el del output **como declaración** (`^ *Nombre:`), no
como texto suelto. El `^ *` y los `:` importan: sin ellos el grep también matchea los nombres que
aparecen dentro de comentarios y te da un falso positivo.
**Salida esperada:** *nada* (exit 1). Si imprime algo, la sesión **ya está pegada** → no la pegues de
nuevo, saltá al Paso 4.

Un logical ID repetido no es un error de YAML: YAML se queda en silencio con el último y `cfn-lint`
tira `E0000 ... found duplicate key`. Si además los dos bloques difieren, desplegás el que no querías.

### Paso 2 — Pegar el recurso en ANCLA 1

```bash
grep -n "ANCLA 1" template.yaml        # te da el número de línea exacto
```

Abrí los dos archivos, copiá del snippet **solo las líneas que empiezan con 2 espacios** (del
`  TranslateCatalogFunction:` hasta la última línea indentada) y pegalas **justo antes** de la caja
`ANCLA 1`, separadas con una línea en blanco y su encabezado de sesión:

```yaml
  # ── S04 · Amazon Translate ─────────────────────────────────────────────────
  TranslateCatalogFunction:
    Type: AWS::Serverless::Function
    ...
          AllowHeaders: [ "*" ]

  # ╔══════════════════════════════════════════════════════════════════════════╗
  # ║  ANCLA 1 — PEGAR ACÁ EL RECURSO DEL PRÓXIMO SNIPPET ...
```

> **La indentación es lo único que puede romper el deploy.** Si el editor te "ayuda" y el
> `TranslateCatalogFunction:` queda en la columna 0, CloudFormation lo lee como clave de nivel raíz
> del template (al mismo nivel que `Resources:`) y el changeset muere con
> `Invalid template property or properties [TranslateCatalogFunction]`. Es exactamente el error del
> deploy anterior. Verificalo con `grep -n "^TranslateCatalogFunction" template.yaml` → debe dar vacío.

### Paso 3 — Pegar el output en ANCLA 2

Descomentá el Output del pie del snippet, dejalo con **2 espacios** y agregale `Description:`:

```yaml
  TranslateCatalogUrl:
    Description: S4 Function URL - catalogo multilingue (Translate)
    Value: !GetAtt TranslateCatalogFunctionUrl.FunctionUrl
```

> **El nombre del `!GetAtt` no es intuitivo:** SAM crea el recurso `<LogicalId>Url` a partir de
> `FunctionUrlConfig`, así que se escribe `TranslateCatalogFunction` + `Url` + `.FunctionUrl`. Si
> ponés `!GetAtt TranslateCatalogFunction.FunctionUrl` el lint falla con
> `Attribute "FunctionUrl" ... does not exist`.

### Paso 4 — Validar antes de gastar un deploy

```bash
sam validate --lint -t template.yaml
```

**Qué hace:** expande el Transform SAM y corre `cfn-lint` sin tocar AWS. Gratis, 3 segundos.
**Salida esperada:** `/workshop/capstone/template.yaml is a valid SAM Template`

Chequeo estructural extra que detecta los dos errores clásicos de un solo golpe:

```bash
grep -c "^Outputs:" template.yaml              # debe imprimir 1
grep -n "^[A-Z][A-Za-z]*Function:" template.yaml   # debe imprimir NADA (nada en columna 0)
```

### Paso 5 — Desplegar

```bash
sam build && sam deploy
```

**Qué hace:** `build` copia cada `CodeUri` a `.aws-sam/build/` e instala dependencias; `deploy` sube
el paquete, crea un changeset y lo ejecuta. Las capabilities y el stack salen de `samconfig.toml`
(`stack_name = "techmoda-ai-li"`), por eso no hay que pasar flags.

**Salida esperada:** `CREATE_COMPLETE` de `TranslateCatalogFunction` + `TranslateCatalogFunctionUrl`
+ el rol `...-TranslateCatalogFunctionRole-...`, y al final la tabla de Outputs con
`TranslateCatalogUrl`.

### Paso 6 — Probar

```bash
URL=$(aws cloudformation describe-stacks --stack-name techmoda-ai-li --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='TranslateCatalogUrl'].OutputValue" --output text)

PID=$(aws dynamodb scan --table-name techmoda-ai-li-Products --region us-east-1 \
  --max-items 1 --query 'Items[0].productId.S' --output text)

curl -s -X POST "${URL%/}/products/$PID/translate" \
  -H 'Content-Type: application/json' -d '{"target":"en"}' | python3 -m json.tool
```

`${URL%/}` **quita la barra final**: las Function URLs siempre terminan en `/` y sin esto la ruta
queda `.../products/...` con doble barra. El `curl` exacto de cada sesión está en su `GUIA.md`.

Después de cada sesión, actualizá el **inventario** de la cabecera de `template.yaml` (líneas 8–15) y
cambiá el "siguiente: SNN" de las dos anclas. Es el único mantenimiento manual y es lo que evita
repegar.

---

## 3. Ficha de cada sesión pendiente

Datos sacados de cada `template-snippet.yaml`. Los logical IDs son **literales**: usalos tal cual en
el `grep` del Paso 1.

### S04 · Amazon Translate — catálogo multilingüe
| | |
|---|---|
| Snippet | [sessions/S04-translate-multilang/template-snippet.yaml](sessions/S04-translate-multilang/template-snippet.yaml) |
| Recursos a ANCLA 1 | `TranslateCatalogFunction` |
| Outputs a ANCLA 2 | `TranslateCatalogUrl` |
| Permiso nuevo | `translate:TranslateText` con `Resource: "*"` (Translate no admite ARN de recurso) |
| Prerequisito | ninguno más allá de S00 |
| Prueba | `POST ${URL%/}/products/<PID>/translate` con `{"target":"en"}` |

### S05 · Amazon Polly — descripción por voz
| | |
|---|---|
| Snippet | [sessions/S05-polly-voice/template-snippet.yaml](sessions/S05-polly-voice/template-snippet.yaml) |
| Recursos a ANCLA 1 | **dos**: `AudioBucket` (S3) **y** `SynthesizeVoiceFunction` |
| Outputs a ANCLA 2 | `SynthesizeVoiceUrl` |
| Permisos nuevos | `polly:SynthesizeSpeech` + `S3CrudPolicy` sobre `AudioBucket` |
| Ojo | primera sesión que agrega un **bucket**. `AudioBucket` es privado (todo el `PublicAccessBlock` en `true`): el mp3 se sirve con **URL prefirmada**, no público. Trae `LifecycleConfiguration` que borra el audio a los 7 días (FinOps) |
| Ojo | al borrar el stack, un `AudioBucket` con objetos deja el delete en `DELETE_FAILED` → `bash scripts/fix-failed-delete.sh` |
| Prueba | `POST ${URL%/}/products/<PID>/voice` con `{"lang":"es"}` |

### S06 · Amazon Bedrock — generar descripciones
| | |
|---|---|
| Snippet | [sessions/S06-bedrock-descripciones/template-snippet.yaml](sessions/S06-bedrock-descripciones/template-snippet.yaml) |
| Recursos a ANCLA 1 | `GenerateDescriptionFunction` |
| Outputs a ANCLA 2 | `GenerateDescriptionUrl` |
| Permiso nuevo | `bedrock:InvokeModel` **acotado por ARN** (`foundation-model/*` + `inference-profile/*`) — Bedrock sí admite ARN, a diferencia de Rekognition/Comprehend/Translate |
| **Prerequisito externo** | habilitar el modelo en **consola → Bedrock → Model access**, en `us-east-1`. Es un setting **por región** y no se puede hacer por template. Si falta: `AccessDeniedException` — ese es el primer sospechoso, no las policies |
| Ojo | `Timeout: 60` sobreescribe el global de 30 s: los modelos generativos tardan segundos |
| Ojo | `BEDROCK_MODEL_ID: anthropic.claude-haiku-4-5-20251001-v1:0` — `bedrock-runtime` exige el ID **versionado**; el alias corto `claude-haiku-4-5` lo rechaza |
| Prueba | `POST ${URL%/}/products/<PID>/describe` con `{"tone":"elegante","save":true}` |

### S07 · Bedrock embeddings — búsqueda semántica / RAG
| | |
|---|---|
| Snippet | [sessions/S07-bedrock-rag-busqueda/template-snippet.yaml](sessions/S07-bedrock-rag-busqueda/template-snippet.yaml) |
| Recursos a ANCLA 1 | **dos**: `IndexEmbeddingsFunction` y `SemanticSearchFunction` |
| Outputs a ANCLA 2 | **dos**: `IndexEmbeddingsUrl` y `SemanticSearchUrl` |
| Permisos | el indexador escribe (`DynamoDBCrudPolicy`), el buscador **solo lee** (`DynamoDBReadPolicy`) — la asimetría es el punto pedagógico, no la copies mal |
| Prerequisito | Model access de `amazon.titan-embed-text-v2:0` en `us-east-1` |
| Orden de uso | **primero** `POST $INDEX_URL` (construye el índice, `Timeout: 120` porque recorre todo el catálogo), **después** `GET ${SEARCH_URL%/}/search?q=...` |

### S08 · Bedrock chatbot RAG — asistente de compras
| | |
|---|---|
| Snippet | [sessions/S08-bedrock-chatbot/template-snippet.yaml](sessions/S08-bedrock-chatbot/template-snippet.yaml) |
| Recursos a ANCLA 1 | `ShoppingAssistantFunction` |
| Outputs a ANCLA 2 | `ShoppingAssistantUrl` |
| Prerequisito | **S07 pegado y su indexador ya corrido** — sin embeddings en DynamoDB el asistente no tiene de dónde recuperar |
| Ojo | necesita `InvokeModel` sobre **dos** modelos (`EMBED_MODEL_ID` para recuperar + `BEDROCK_MODEL_ID` para generar); el ARN comodín del snippet ya cubre los dos |
| Prueba | `POST $URL` con `{"message":"busco un abrigo para el invierno"}` |

### S09 · Guardrails y sesgo — **no toca `template.yaml`**
| | |
|---|---|
| Archivos | [sessions/S09-guardrails-sesgo/guardrail-config.json](sessions/S09-guardrails-sesgo/guardrail-config.json) + [create-guardrail.sh](sessions/S09-guardrails-sesgo/create-guardrail.sh) |
| Qué hacer | `bash sessions/S09-guardrails-sesgo/create-guardrail.sh` — el guardrail se crea por **CLI**, no por CloudFormation |
| Ojo | no busques snippet: no existe. Nada que pegar en ninguna ancla |

### S10 · IAM, logging y costos — gobernanza
| | |
|---|---|
| Snippet | [sessions/S10-iam-logging-costos/template-snippet-governance.yaml](sessions/S10-iam-logging-costos/template-snippet-governance.yaml) (nombre distinto: `-governance`) |
| Recursos a ANCLA 1 | `AiCostAlarm` (CloudWatch Alarm) y `BedrockInvocationLogGroup` (Log Group) |
| Outputs a ANCLA 2 | **ninguno** — no hay Lambda ni Function URL en esta sesión |
| Tercera parte | el pie del snippet pide agregar `Tags:` **en `Globals.Function`**, no en `Resources:`. Es el único snippet que toca `Globals:` (líneas 40–51 del template): pegalo indentado 4 espacios bajo `Function:` |
| Ojo | las métricas de `AWS/Billing` existen **solo en us-east-1**, que ya es la región del stack. Si el sandbox no deja crear alarmas, creala a mano o usá AWS Budgets |
| Manual después | los cost allocation tags hay que **activarlos** en Billing → Cost Allocation Tags para verlos en Cost Explorer (tarda hasta 24 h en aparecer) |

### S11 · Integración, demo y cleanup — no toca `template.yaml`
`bash sessions/S11-integracion-demo-cleanup/demo.sh` recorre todos los endpoints. El cleanup es
`bash scripts/delete-all.sh`.

---

## 4. Cómo saber en cualquier momento qué está pegado y qué está desplegado

```bash
# 1) Qué recursos y outputs declara el template (lo que pegaste)
grep -nE '^  [A-Z][A-Za-z0-9]*:$' template.yaml

# 2) Qué está realmente en AWS (lo que desplegaste)
aws cloudformation describe-stacks --stack-name techmoda-ai-li --region us-east-1 \
  --query 'Stacks[0].Outputs[].OutputKey' --output text

# 3) Estado general del stack + Function URLs + conteo de productos
bash scripts/status.sh
```

Si (1) tiene algo que (2) no: falta `sam deploy`. Si (2) tiene algo que (1) no: pegaste sobre un
template viejo — **no despliegues**, o CloudFormation borra el recurso que falta.

### Dos gotchas de scripts que te van a confundir

**`validate-all.sh` sección 3 reporta S01/S02/S03 como FALLO. Es esperado, no es una regresión.**
Esa sección simula al alumno: agarra cada snippet y lo **inserta** en tu `template.yaml` para
lintear. Como S01–S03 ya están pegados, la inserción crea logical IDs duplicados. Cada sesión que
pegues suma un "fallo" más ahí. Lo que importa son las secciones 1, 2 y 4.

**`$STACK_NAME` del entorno apunta a otro stack.** `/etc/profile.d/techmoda-env.sh` exporta
`STACK_NAME=techmoda-mxmex35-lisandrounam`, mientras que `samconfig.toml` dice `techmoda-ai-li`.
Resultado: `sam deploy` va al stack correcto, pero `validate-all.sh`, `status.sh` y `bootstrap.sh`
inspeccionan el otro (que está vacío). Antes de correr scripts:

```bash
export STACK_NAME=techmoda-ai-li
```

---

## 5. Tabla de fallas

| Síntoma | Causa | Arreglo |
|---|---|---|
| `Invalid template property or properties [XxxFunction]` al crear el changeset | el recurso quedó en la **columna 0** → CloudFormation lo lee como clave de nivel raíz | indentarlo 2 espacios dentro de `Resources:`. Detectalo antes con `grep -n "^[A-Z][A-Za-z]*Function:" template.yaml` |
| El deploy sale OK pero **desaparecieron** `ApiUrl` / `FrontendUrl` | pegaste un **segundo** `Outputs:`; YAML conserva el último | dejar un solo bloque. `grep -c "^Outputs:" template.yaml` → 1 |
| `E0000 ... found duplicate key "XxxFunction"` | pegaste dos veces la misma sesión | borrar la copia. Prevenir con el `grep` del Paso 1 |
| `Attribute "FunctionUrl" ... does not exist` | escribiste `!GetAtt XxxFunction.FunctionUrl` | va `!GetAtt XxxFunctionUrl.FunctionUrl` (SAM crea el recurso `<LogicalId>Url`) |
| `sam deploy` dice *No changes to deploy* pero falta la función | desplegaste otro template (`-t template.full.yaml`) o el otro stack | `sam deploy` sin `-t`; `export STACK_NAME=techmoda-ai-li` |
| `is not authorized to perform: iam:CreateRole` | falta el `PermissionsBoundary` de `Globals` | no lo borres: la policy de la cuenta solo permite crear roles con ese boundary |
| `AccessDeniedException` en S06/S07/S08 | **Model access** de Bedrock no habilitado en `us-east-1` | consola → Bedrock → Model access. No es IAM |
| 404 / *model not found* en Bedrock | `BEDROCK_MODEL_ID` sin versión, o falta el prefijo `us.` del inference profile | usar el ID versionado del snippet |
| `TypeError: Float types are not supported` | un `Confidence` de Rekognition fue a DynamoDB como `float` | `Decimal(str(v))` antes del `update_item` |
| Ruta con `//` o 404 al invocar | te olvidaste `${URL%/}` (la Function URL termina en `/`) | usar `${URL%/}/products/...` |
| `sam build` falla con *CodeUri not found* | corriste desde otro directorio | los `CodeUri` son relativos al template: ejecutar desde `/workshop/capstone` |
| Los cost allocation tags de S10 no salen en Cost Explorer | falta activarlos en Billing | Billing → Cost Allocation Tags; tarda hasta 24 h |

---

## 6. Qué evalúa el examen (AIF-C01) en esto

Aunque pegar YAML parece plomería, el patrón está diseñado alrededor de puntos del examen:

- **Elegir el servicio de IA correcto por modalidad** (D1/D2) — es literalmente el orden de las
  sesiones: imagen → **Rekognition** (S01/S02), texto/sentimiento → **Comprehend** (S03), idioma →
  **Translate** (S04), voz → **Polly** (S05), generación → **Bedrock** (S06), embeddings/RAG →
  **Bedrock + Titan** (S07/S08).
- **Inferencia vs. entrenamiento** — todo el capstone hace solo **inferencia** sobre modelos
  administrados y preentrenados. No hay dataset, ni GPUs, ni SageMaker.
- **Mínimo privilegio, y sus dos formas** (D5) — cuando el servicio **admite ARN de recurso**
  (Bedrock, S3, DynamoDB) se acota por **recurso**; cuando **no lo admite** (las APIs `Detect*` de
  Rekognition, `Comprehend`, `Translate`) el control correcto es acotar la **acción** exacta con
  `Resource: "*"`. `rekognition:*` sería la respuesta incorrecta en el examen.
- **RAG** (D3/D4) — S07/S08: recuperar contexto de una fuente propia y pasárselo al modelo, en vez de
  reentrenarlo. Es la alternativa barata al fine-tuning.
- **Guardrails y IA responsable** (D4) — S09: filtros de contenido y sesgo aplicados *fuera* del
  modelo.
- **Gobernanza, observabilidad y FinOps** (D5) — S10: tags de asignación de costos, alarmas de
  presupuesto, retención de logs acotada. Y el `ExpirationInDays: 7` del bucket de audio de S05.

## 7. Costo y cleanup

Todos los servicios de estas sesiones se cobran **por uso** (por imagen, por carácter, por token): un
par de docenas de invocaciones de práctica son **centavos**. *Verificar las cifras en las páginas de
precios oficiales.* Lo que **sí** cuesta si queda encendido: CloudFront, los buckets con objetos y la
tabla DynamoDB.

```bash
bash scripts/delete-all.sh          # pide confirmación explícita ("si"); borra el stack completo
bash scripts/fix-failed-delete.sh   # si quedó en DELETE_FAILED por buckets no vacíos
```

Detalle por sesión en el bloque de cleanup de cada `GUIA.md` y en
[docs/COST_AND_CLEANUP.md](docs/COST_AND_CLEANUP.md).
