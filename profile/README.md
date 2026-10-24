# Parse Documents Fast

Plataforma de microservicios orientada a la recepción, validación, procesamiento y conversión de documentos PDF. El sistema extrae el contenido de un PDF a Markdown —el formato canónico de trabajo— y permite descargar los documentos nuevamente en PDF, según se requiera.

La arquitectura está compuesta por servicios independientes, desplegados en contenedores y coordinados a través de un punto de entrada único. Cada servicio tiene una responsabilidad acotada y se comunica con los demás mediante contratos estables.

## Capacidades

Los repositorios de la organización se organizan por responsabilidad:

- **pdf-infra** — Infraestructura transversal del sistema. Proporciona el punto de entrada único mediante un reverse proxy y gateway de API, junto con los componentes de soporte: limitación de tráfico, colas de trabajo, almacenamiento en caché y base de datos.
- **pdf-main** — Punto de entrada de la API pública y orquestador principal. Recibe las solicitudes, coordina la validación, el encolado y la persistencia, y devuelve las respuestas y estados al cliente.
- **pdf-validator** — Servicio ligero y síncrono de validación perimetral. Verifica el tipo y la integridad de los archivos recibidos antes de que ingresen al flujo de procesamiento.
- **pdf-extractor** — Servicio de procesamiento asíncrono e intensivo en cómputo. Convierte documentos PDF a Markdown de forma estructurada, gestionando la carga mediante un grupo de trabajadores con control de presión.
- **pdf-persistence** — Servicio de persistencia. Almacena los metadatos, el estado y los resultados de cada documento procesado, y resuelve la detección de documentos duplicados.
- **pdf-converter** — Servicio de conversión de Markdown a PDF mediante Pandoc. Opera de forma síncrona únicamente en el momento de la descarga, sin persistir archivos intermedios en disco.

## API pública

La API se expone a través del punto de entrada `pdf-main`, detrás del gateway. Los siguientes son los endpoints disponibles:

| Método | Ruta | Descripción |
| --- | --- | --- |
| `POST` | `/api/pdfs` | Sube un documento (PDF o Markdown) para su procesamiento. |
| `GET` | `/api/pdfs` | Lista los documentos registrados. |
| `GET` | `/api/pdfs/{id}` | Obtiene un documento con su contenido. |
| `DELETE` | `/api/pdfs/{id}` | Elimina un documento. |
| `GET` | `/api/pdfs/{id}/download?format=markdown\|pdf` | Descarga un documento en Markdown (por defecto) o PDF. |
| `POST` | `/extract` | Extracción síncrona directa, reservada para pruebas de rendimiento. |

Los documentos PDF se procesan de forma asíncrona: al subirlos se devuelve un estado inicial y el resultado queda disponible una vez completada la extracción. Los documentos Markdown se persisten de forma inmediata, dado que ya se encuentran en el formato canónico.

## Infraestructura y protecciones

El acceso al sistema se realiza a través de un único gateway que aplica las siguientes protecciones:

- **Terminación de tráfico cifrado**: el gateway gestiona las conexiones entrantes y enruta las solicitudes a los servicios internos.
- **Limitación de velocidad**: cada cliente tiene un máximo de solicitudes por segundo, calculado de forma distribuida para impedir abusos.
- **Cortacircuitos**: ante una degradación sostenida de un servicio, el sistema interrumpe temporalmente el tráfico hacia el mismo para evitar fallos en cascada y permitir su recuperación.
- **Aislamiento de componentes de soporte**: las colas de trabajo, la caché y el control de velocidad operan en instancias separadas, de modo que la saturación de una no afecta a las demás.
- **Base de datos única y compartida**: una sola instancia de almacenamiento es consumida por el servicio de persistencia, con índice de unicidad que impide la duplicación de documentos.

## Modo de evaluación de rendimiento

Para ejecutar el sistema en modo de evaluación de rendimiento es necesario utilizar los archivos `docker-compose.tp.yml`, tanto el de la raíz como los de cada servicio, en lugar de la configuración estándar.

Este modo aplica límites fijos de recursos por réplica, escala horizontalmente los servicios y desactiva temporalmente el control de velocidad y el cortacircuitos, con el objetivo de medir el rendimiento máximo de la plataforma. Como consecuencia, no refleja el comportamiento esperado en un entorno de producción.

Los resultados y las consideraciones detalladas sobre el uso de este modo opcional se documentan en el archivo `README.md` ubicado en la raíz del proyecto.

