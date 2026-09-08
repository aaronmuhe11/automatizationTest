# Restful Booker – Suite de pruebas de API automatizadas

Suite de pruebas automatizadas sobre la API pública [Restful Booker](https://restful-booker.herokuapp.com/apidoc/index.html), que simula el sistema de reservaciones de un hotel.

El proyecto cubre el ciclo de vida completo de una reservación (crear, consultar, actualizar, eliminar y verificar la eliminación), con ejecución dirigida por datos, ejecución desde línea de comandos e integración a un pipeline de CI.

**Stack:** Postman · Newman · Jenkins · JavaScript

---

## Última ejecución

| Métrica | Ejecutadas | Fallidas |
|---------|-----------:|---------:|
| Iteraciones | 5 | 0 |
| Requests | 40 | 0 |
| Test scripts | 40 | 0 |
| Pre-request scripts | 15 | 0 |
| **Aserciones** | **120** | **0** |

Duración total: 9.1 s · Tiempo de respuesta promedio: 127 ms (mín. 111 ms, máx. 627 ms)

El reporte HTML completo de esta corrida está en [`reports/resultado.html`](reports/resultado.html).

---

## Qué valida esta suite

| # | Request | Método | Qué comprueba |
|---|---------|--------|---------------|
| 00 | Verificar API | GET | Health check: la API está disponible antes de correr el flujo |
| 01 | Crear token | POST | Autenticación y obtención de un token válido |
| 02 | Crear reservación | POST | Creación correcta y que los datos guardados coincidan con los enviados |
| 03 | Consultar reservación | GET | **Persistencia**: lo consultado es idéntico a lo que se creó |
| 04 | Actualizar completa | PUT | Reemplazo total del recurso, formato y tiempo de respuesta |
| 05 | Actualizar parcial | PATCH | Se modifican solo los campos enviados y **se conservan los demás** |
| 06 | Eliminar reservación | DELETE | Eliminación del recurso |
| 07 | Validar eliminación | GET | El recurso ya no existe (404) |

---

## Decisiones de diseño

**Encadenamiento entre requests.**
El token generado en `01` y el `bookingid` generado en `02` se guardan en variables de entorno y alimentan a los requests siguientes. La suite se autoabastece: no requiere copiar credenciales ni IDs a mano entre corridas.

**Validación de persistencia, no solo de respuesta.**
En `02` se almacena el objeto completo de la reservación creada en la variable `booking_esperado`. En `03` se compara la respuesta del GET contra ese objeto. Un código 200 en el POST solo confirma que el servidor recibió la petición; esta comparación confirma que el dato realmente quedó guardado y se recupera íntegro.

**Aserciones contra el payload real, no contra valores fijos.**
Los tests de `02`, `04` y `05` comparan la respuesta contra el cuerpo efectivamente enviado, resolviendo las variables en tiempo de ejecución:

```javascript
const datosEnviados = JSON.parse(pm.variables.replaceIn(pm.request.body.raw));
pm.expect(respuesta.booking).to.eql(datosEnviados);
```

De esta forma los tests siguen siendo válidos con cualquier juego de datos de entrada, sin necesidad de editarlos cuando cambia el CSV.

**Validación de regla de negocio en el PATCH.**
El test *"Los campos no modificados se conservan"* comprueba que una actualización parcial no destruya el resto del registro. Es el caso que distingue un `PATCH` correctamente implementado de uno que se comporta como `PUT`, y el tipo de defecto que un test de estructura no detectaría.

**Datos variables por diseño.**
Los datos de identidad y fechas provienen del CSV; el precio y el estado del depósito se generan aleatoriamente en pre-request scripts. Esto amplía la cobertura de combinaciones sin necesidad de mantener más filas en el archivo de datos.

---

## Estructura del proyecto

```
Newman/
├── Restful_Booker-CRUD.postman_collection.json      Colección con los 8 requests y sus tests
├── Entorno_Restful_Booker.postman_environment.json  Variables de entorno
├── MOCK_DATA_BOOKER.csv                             Datos de prueba (5 registros)
└── reports/
    └── resultado.html                               Reporte de la última ejecución
```

### Variables de entorno

| Variable | Uso |
|----------|-----|
| `base_url` | URL base de la API |
| `auth_token` | Token generado en el login, consumido por PUT, PATCH y DELETE |
| `booking_id` | ID de la reservación creada, usado por los requests posteriores |
| `booking_esperado` | Objeto completo de la reservación, para validar persistencia |

Solo `base_url` tiene valor inicial. Las otras tres se pueblan en tiempo de ejecución mediante `pm.environment.set()`.

> **Nota sobre valores en Postman.** El valor exportado en el archivo de entorno es el *Shared Value* (antes *Initial value*); el *Value* local no viaja en el export. Por eso las credenciales y tokens deben mantenerse solo en el valor local, para que no terminen versionados en el repositorio.

---

## Cómo ejecutar

### Desde Postman

1. Importar la colección: **File → Import** → seleccionar el archivo `.postman_collection.json`
2. Importar el entorno por separado: **Environments → Import**
3. Activar el entorno *Entorno Restful Booker*
4. Ejecutar los requests en orden, o correr la colección desde el **Collection Runner**

> El uso de un archivo de datos (CSV) en el Collection Runner requiere un plan de pago de Postman. La ejecución dirigida por datos de este proyecto se hace con Newman, que no tiene esa restricción y además es la forma en que corre dentro del pipeline.

### Desde línea de comandos con Newman

Requiere Node.js instalado.

```bash
npm install -g newman newman-reporter-htmlextra
```

```bash
newman run Restful_Booker-CRUD.postman_collection.json \
  -e Entorno_Restful_Booker.postman_environment.json \
  -d MOCK_DATA_BOOKER.csv \
  -r cli,htmlextra \
  --reporter-htmlextra-export reports/resultado.html
```

| Parámetro | Función |
|-----------|---------|
| `-e` | Archivo de entorno |
| `-d` | Archivo de datos: genera una iteración por cada fila |
| `-r cli,htmlextra` | Reporters activos: consola y reporte HTML |
| `--reporter-htmlextra-export` | Ruta de salida del reporte |

Con el CSV incluido, cada corrida ejecuta 5 iteraciones completas del flujo.

---

## Integración con Jenkins

La suite se ejecuta desde un job Freestyle que clona este repositorio, corre Newman y publica el reporte HTML como enlace navegable dentro del propio job.

**Plugins requeridos:** NodeJS, HTML Publisher, AnsiColor, Git

**Configuración resumida:**

1. *Manage Jenkins → Tools → NodeJS*: agregar una instalación LTS y en *Global npm packages to install* declarar `newman newman-reporter-htmlextra`
2. En el job, habilitar *Provide Node & npm bin/folder to PATH*
3. *Source Code Management*: apuntar a este repositorio con una credencial de GitHub (usuario + Personal Access Token)
4. *Build Step* de tipo shell/batch con el comando de Newman
5. *Post-build Actions → Publish HTML reports*, con directorio `reports` e índice `resultado.html`

> **Nota:** Jenkins aplica por defecto una Content Security Policy restrictiva que impide que el reporte de htmlextra cargue sus estilos y gráficas. Es necesario relajarla mediante la propiedad `hudson.model.DirectoryBrowserSupport.CSP` para que el reporte se visualice correctamente.

---

## Hallazgos sobre la API bajo prueba

Los tests validan el comportamiento **observado** de Restful Booker, no el esperado según convención. Durante la construcción de la suite se identificaron dos desviaciones que en un entorno real se reportarían:

- `GET /ping` responde **201 Created**, cuando un health check debería responder **200 OK**.
- `DELETE /booking/{id}` responde **201 Created**, cuando lo convencional sería **200 OK** o **204 No Content**.

Ambos comportamientos están reflejados tal cual en las aserciones, para que la suite refleje la realidad del sistema y no una expectativa.

---

## Limitaciones conocidas

**Los requests no son independientes.** La suite está diseñada como un flujo end-to-end: cada request depende del estado que generó el anterior, por lo que debe ejecutarse completa y en orden. Ejecutar un request aislado falla.

Es un trade-off asumido: se gana el modelado fiel de un caso de uso real, se pierde la posibilidad de ejecución en paralelo y el aislamiento entre casos. En una suite de mayor tamaño convendría dotar a cada caso de su propio setup.

**Dependencia de un servicio público.** Restful Booker está alojado en un entorno gratuito y puede presentar latencia o indisponibilidad intermitente. Por eso los umbrales de tiempo de respuesta se fijaron en 2000 ms: es un margen holgado para evitar falsos negativos por latencia del hosting, no un objetivo de performance.

---

## Próximos pasos

- Migrar la ejecución a GitHub Actions como alternativa a Jenkins
- Añadir validación de schema con JSON Schema
- Incorporar casos negativos: credenciales inválidas, payloads mal formados, IDs inexistentes
- Dotar de setup propio a los requests para permitir ejecución aislada

---

## Autor

**Aarón Muñoz** — QA · ISTQB Foundation Level
