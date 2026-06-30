# Contexto del Sistema: Librerías y Módulos Estándar (OTIM / TGSS)

Este documento sirve como base de conocimiento y guía de referencia para el agente de Copilot. Contiene la especificación técnica, convenciones, arquitecturas y ejemplos de uso de los módulos centrales desarrollados por la OTIM para la estandarización de las APIs, validaciones de dominio, impresión orientada a SOAP y utilidades generales.

---

## 1. Módulo de Paginación y Filtros (`lib-modules-pagination`)

### Descripción General
Desarrollado por la OTIM para estandarizar y facilitar la paginación, ordenación y filtrado en la exposición de APIs, asegurando una estructura uniforme en las peticiones y respuestas de los microservicios.

### Definición de parámetros en OpenAPI (YAML)
Al diseñar un endpoint, se deben incluir los siguientes parámetros de consulta (*query parameters*) según las necesidades del caso de uso (filtros, ordenación, paginación o combinados):

```yaml
parameters:
  - in: query
    name: filters
    schema:
      type: string
      pattern: [ ^<> ]
    description: Fields to filter the search
  - in: query
    name: sorts
    schema:
      type: string
      pattern: [ ^<> ]
    description: Fields to sort the result
  - in: query
    name: pageNumber
    schema:
      type: integer
      format: int32
    description: The page number (Estándar Backend: 0 a n-1)
  - in: query
    name: pageSize
    schema:
      type: integer
      format: int32
    description: The number of elements in the result
```

### Formato de Parámetros Especiales

| Parámetro | Uso y Descripción | Formato | Ejemplo |
| :--- | :--- | :--- | :--- |
| `filters` | Recibe los criterios de filtrado con soporte de operadores lógicos (`AND`, `OR`). | `campo:operador:valor` | `nombre:EQ:"Pedro" OR apellido:LIKE:"san" AND sueldo:GT:3000` |
| `sorts` | Define los campos de ordenación y su dirección. | `campo1,campo2-,campo3` | `nombre,sueldo-` *(sueldo descendente)* |

#### Consideraciones Críticas para el Agente:
* **Excepciones de Formato:** Si `filters` o `sorts` no cumplen la sintaxis exacta, el sistema lanzará una excepción de validación.
* **Dirección del Sort:** El orden ascendente requiere solo el nombre del campo (ej. `nombre`). El orden descendente se indica con un signo menos posfijo (ej. `sueldo-`).
* **Restricción de Caracteres:** No se permiten caracteres especiales fuera del patrón. Problemas con nombres de campos específicos deben escalarse a la OTIM.
* **Operadores de Filtro Disponibles:**
  * `EQ` (Igual), `NE` (No igual)
  * `LT` (Menor que), `GT` (Mayor que), `LE` (Menor o igual), `GE` (Mayor o igual)
  * `LIKE` (Contiene texto / como)
  * `IN` (Incluido en conjunto), `NOT_IN` (No incluido en conjunto)
  * `CONTAINS` (Contiene elemento), `NOT_CONTAINS` (No contiene)
  * `CONTAINS_KEY` (Contiene clave), `NOT_CONTAINS_KEY` (No contiene clave)

### Gestión de la Request en Java / Spring

El punto de entrada (*Controller* o *Delegate*) recibe los parámetros encapsulados en objetos `Optional`.

```java
@Override
public ResponseEntity<?> metodoPrueba(
    final Optional<String> filters,
    final Optional<String> sorts,
    final Optional<Integer> pageNumber,
    final Optional<Integer> pageSize) {
    
    // Uso del Builder GissRequest
    GissRequest request = GissRequest.create()
        .withFilters(filters)
        .withPagination(sorts, pageNumber, pageSize);
}
```

#### Valores por Defecto de `GissRequest`:
* `filters`: Ninguno (no se aplican filtros).
* `sorts`: `Sort.unsorted()` (sin ordenación).
* `pageNumber`: `0` (primera página).
* `pageSize`: `10` (diez elementos).

#### Reglas de Validación de Paginación (Excepciones):
* `pageNumber` < 0 lanza excepción.
* `pageSize` < 1 lanza excepción (mínimo 1 elemento).
* `pageSize` > 50 lanza excepción (máximo permitido por página).

> **Nota de Arquitectura:** El frontend maneja índices de página basados en `1-n`, mientras que el backend sigue el estándar de *Spring Data Rest* (`0` a `n-1`). La traducción e implementación dependerá del framework de persistencia subyacente (JPA, Hibernate, SQL Nativo, MongoDB, Criteria, etc.).

### Implementación del Filtrado con QueryDSL
`GissRequest` abstrae la construcción de predicados QueryDSL (compatible con MongoDB, JDBC RawSQL, JPA, etc.). Se debe mapear la clave recibida en la API con el path de la QEntity correspondiente:

```java
@Override
public ResponseEntity<?> metodoPrueba(final Optional<String> filters) {
    GissRequest request = GissRequest.create().withFilters(filters);
    QGandalf qgandalf = QGandalf.gandalf;
    
    Map<String, Path<?>> paths = new HashMap<>() {{
        put("nombre", qgandalf.nombre);
        put("mejor_que", qgandalf.mejorQue);
        put("edad", qgandalf.edad);
    }};
    
    // Generación automática del Predicate para la BBDD
    BooleanBuilder booleanExpression = request.obtainFilters(paths);
}
```

#### Inyección Dinámica de Filtros (Backend-side):
Para añadir reglas lógicas de negocio adicionales desde el backend sin alterar la petición del cliente, se usan `appendFilters` o `prependFilters` **antes** de invocar `obtainFilters`:

```java
request.appendFilters(FiltersExpressionValue.builder()
    .field("precio")
    .operator(FiltersExpressionOperators.GE)
    .operand("200")
    .build(), Operators.AND);
```
*Nota: El nuevo campo inyectado obligatoriamente debe existir en el mapa de `paths`.*

### Ejemplos Avanzados de Consultas Estructuradas
* **Múltiples elementos por el mismo atributo (Uso de OR):**
  `id:EQ:"05efc014-dfc5-c559-e063-045bc70a7533" OR id:EQ:"93492b48-ff5f-4820-abc0-fcc6a7ab1998"`
* **Propiedad Distributiva para agrupaciones complejas:**
  Sabiendo que `(A OR B) AND C AND D -> (A AND C AND D) OR (B AND C AND D)`:
  `id:EQ:"05efc014-dfc5-c559-e063-045bc70a7533" AND activado:EQ:"true" AND entornos:CONTAINS:"INT" OR id:EQ:"93492b48-ff5f-4820-abc0-fcc6a7ab1998" AND activado:EQ:"true" AND sistema:CONTAINS:"INT"`

---

## 2. Módulo de Dominio y Validaciones (`lib-modules-domain-tgss`)

Proporciona Value Objects autónomos y validadores para garantizar la integridad de los datos de negocio sectoriales y comunes.

### 2.1 Bancos: IBAN
* **Clase:** `es.segsocial.tg.cnac.libc.tgss.domain.validation.bancos.Iban`
* **Uso:** Factoría estática `Iban.of(String iban)`. Valida la estructura internacional y el dígito de control.
* **Propiedades expuestas:** País, código de control, número de cuenta y el método `.getIbanCompleto()`.

### 2.2 Cuenta de Cotización: CCC
* **Clase:** `es.segsocial.tg.cnac.libc.tgss.domain.validation.ccc.Ccc`
* **Uso:** `Ccc.of(String identificador)` o `Ccc.of(String numero, String regimen)`.
* **Propiedades expuestas:** Régimen, provincia, número secuencial, dígito de control y el método `.getCccCompleto()`.

### 2.3 Datos de Contacto: Teléfono y Email
* **Teléfono:** Utiliza la abstracción de `google/libphonenumber`.
  * **Clase:** `es.segsocial.tg.cnac.libc.tgss.domain.validation.datoscontacto.Telefono`
  * **Factorías:**
    * `Telefono.of(identificador)` -> Región por defecto null.
    * `Telefono.ofDefaultEs(identificador)` -> Fuerza región de España ("ES").
    * `Telefono.of(identificador, defaultRegion)`.
  * **Métodos clave:** `.getInternationalNumber()` (retorna formato E.164, ej: `+34600123456`) e inspección de tipo (Fijo/Móvil) a través del objeto subyacente `PhoneNumber`.
* **Email:** Basado en `apache-commons/validator`.
  * **Clase:** `es.segsocial.tg.cnac.libc.tgss.domain.validation.datoscontacto.Email`
  * **Métodos clave:** `Email.of("user@domain.com")` expone de forma separada las propiedades de `.getCuenta()` y `.getDominio()`, así como `.toFullEmail()`.

### 2.4 Personas Físicas: Identificadores (DNI, NIE, NIF, NUSS)
* **IpfParcial (A11 - 11 caracteres):**
  * `IpfParcial.of(identificador)`. Resuelve automáticamente el `TipoIpf` (Enum). Método `.getTipoAndIpf()` normaliza la salida.
* **Ipf (A15 - 15 caracteres):**
  * `Ipf.of(identificador)`. Descompone de forma estricta: Tipo, Identificador base, Dígito de Duplicidad y Dígito de Desdoblamiento.
* **Enum `TipoIpf` soportados:**
  * `DNI` ("1"), `NIE` ("6"), `NIF` ("9"), `PASAPORTE` ("2"), `IPF` ("0"), y Afiliados No Residentes (`TIPO_K`, `TIPO_L`, `TIPO_M`).
* **NUSS / NAF (A12):**
  * `Nuss.of(identificador)`. Valida la estructura de la Seguridad Social descomponiendo Provincia, Número secuencial y Dígitos de control.

### 2.5 Personas Jurídicas: CIF
* **Validador:** `es.segsocial.tg.cnac.libc.tgss.domain.validation.personasjuridicas.validators.CifValidator`
* **Uso:** Instanciación clásica e invocación del método booleano `.isValid(String cif)`. Soporta adecuadamente la lógica de control numérica y de caracteres alfabéticos de organizaciones (ej: `B12345674`, `P2345678C`).

### 2.6 Territorios: Provincias e ISOs
* **`ComunidadAutonoma`:** Enum parametrizado según estándar `ISO-3166-2:ES`. Proporciona código oficial, número de orden e ID literal (ej: `ES_MD -> "13", 13, "Comunidad de Madrid"`).
* **`Provincia`:** Enum enlazado a su Comunidad Autónoma. Resuelve códigos multi-asignados por la Agencia Tributaria debido al agotamiento de rangos históricos.
  * *Método útil:* `Provincia.fromCodigo(String codigo)` devuelve un `Optional<Provincia>`.

---

## 3. Módulo de Impresión SOAP (`lib-modules-soap-print`)

Habilita clientes automáticos SOAP altamente configurables para interactuar con los servicios centrales de generación e impresión de documentos corporativos.

### Paso 1: Dependencia Maven (POM)
```xml
<dependency>
  <groupId>es.segsocial.tg.cnac.libc</groupId>
  <artifactId>lib-modules-soap-print</artifactId>
  <version>${lib-modules.version}</version>
</dependency>
```

### Paso 2: Propiedades del Entorno (`application-commons.yaml`)
Se deben configurar las URIs de los endpoints de los servicios (Síncronos y Asíncronos):
```yaml
wsclient:
  generador-documento:
    uri: "[http://we9n.des.portal.ss:1089/INFRWSIPSincrono/GeneradorDocumentoService](http://we9n.des.portal.ss:1089/INFRWSIPSincrono/GeneradorDocumentoService)"
  generador-documento-asincrono:
    uri: "[http://we9n.des.portal.ss:1089/INFRWSIPAsincrono/GeneradorDocumentoAsincronoService](http://we9n.des.portal.ss:1089/INFRWSIPAsincrono/GeneradorDocumentoAsincronoService)"
  recuperar-documento-asincrono:
    uri: "[http://we9n.des.portal.ss:1089/INFRWSIPAsincrono/RecuperarDocumentoAsincronoService](http://we9n.des.portal.ss:1089/INFRWSIPAsincrono/RecuperarDocumentoAsincronoService)"
```

### Paso 3: Activación Mediante Anotaciones
El desarrollador debe crear una clase de configuración anotada según el servicio que requiera consumir:

| Servicio Solicitado | Anotación de Activación | Objeto Cliente Inyectable |
| :--- | :--- | :--- |
| **GeneradorDocumento (Síncrono)** | `@EnableGeneradorDocumento` | `GeneradorDocumento` |
| **GeneradorDocumento Asíncrono** | `@EnableGeneradorDocumentoAsincrono` | `GeneradorDocumentoAsincrono` |
| **RecuperarDocumento Asíncrono** | `@EnableRecuperarDocumentoAsincrono` | `RecuperarDocumentoAsincrono` |

*Ejemplo de Configuración de Prioridad:*
```java
@EnableGeneradorDocumentoAsincrono
@Configuration
public class GeneradorDocumentoConfig { }
```

### Paso 4: Catálogo de Métodos Disponibles

#### Servicio Síncrono (`GeneradorDocumento`)
* `getPDFTratamiento`: Genera PDF básico e inmediato sin firmar.
* `getPDFFirmadoTratamiento`: Genera PDF aplicando firma digital homóloga.
* `getWordTratamiento` / `getExcelTratamiento` / `getPPTTratamiento`: Genera formatos ofimáticos estructurados.

#### Servicio Asíncrono (`GeneradorDocumentoAsincrono`)
* Métodos para procesamientos pesados de fondo con retorno diferido:
  * `generarPDFAsincronoTratamientoSeguimiento`
  * `generarWordAsincronoTratamientoSeguimiento`
  * `generarExcelAsincronoTratamientoSeguimiento`
  * `generarPPTAsincronoTratamientoSeguimiento`

#### Servicio de Recuperación (`RecuperarDocumentoAsincrono`)
* `getDocumentoAsincrono`: Descarga o recupera el archivo binario procesado pasándole el ID único de tarea devuelto por el flujo asíncrono.

### Parámetros Esenciales del Método de Impresión
* `aplicacion` (String): ID del sistema emisor registrado en la plataforma de impresión.
* `origen` (String): Filtrado estricto de red, acepta exclusivamente `INTRANET` o `INTERNET`.
* `informe` (String): Código identificativo de la plantilla física dada de alta en el sistema.
* `dataHandler` (Base64Binary/String): Flujo de datos estructurados para popular la plantilla.
* `llevaCEA` (boolean): `true` fuerza la incrustación y renderizado del Código Electrónico de Autenticidad.
* `caducidad` (int): `-100` aplica la caducidad por defecto de la plantilla; `-1` define almacenamiento permanente en el SGDA; valores $\ge 0$ definen días de persistencia en caché (Máximo 733 días).
* `tipoCEA` (String): `T` (Temporal con caducidad) o `P` (Permanente, requiere mapeo en `informacionSGDA`).

---

## 4. Conexión Documental y Expedientes (`lib-modules-soap-sgda`)

Este módulo proporciona la infraestructura necesaria para interactuar de forma nativa con el Gestor Documental Automatizado (SGDA) mediante mensajería SOAP y esquemas WSDL pre-compilados.

### Activación y Mapeo de Beans

1. **Inclusión del Artefacto:** Dependencia en el bloque POM con la firma `lib-modules-soap-sgda`.
2. **Habilitación Selectiva:** Dependiendo de las operaciones del ciclo de vida documental requeridas, se activan los beans correspondientes usando anotaciones a nivel de `@Configuration`:

| Contexto Operativo WSDL | Anotación de Activación | Bean Habilitado en Contexto |
| :--- | :--- | :--- |
| Gestión Integral de Expedientes | `@EnableExpediente` | `OperacionesExpediente` |
| Ingesta y Mutación de Documentos | `@EnableDocumento` | `OperacionesDocumento` |
| Consultas y Búsquedas Indexadas | `@EnableBusqueda` | `OperacionesBusqueda` |

3. **Propiedades de Infraestructura:** Requiere dar de alta la parametrización de conectores en la clave raíz `wsclient` del archivo `application-commons.yaml`. Una vez cargado el contexto, se expone e inyecta de forma segura el bean centralizado `Conexion` para gobernar las credenciales y sesiones.

---

## 5. Módulo de Utilidades Transversales (`lib-modules-utils`)

Capa auxiliar destinada a reducir la duplicidad de código estructurado y centralizar mecánicas recurrentes del ecosistema de desarrollo.

### Arquitectura de Componentes
El módulo está segmentado lógicamente en tres componentes principales:

* **API Utils:** Métodos estáticos de normalización de respuestas HTTP, interceptores de errores comunes y parseadores de cabeceras estándar de comunicación.
* **CnacLogger:** Envoltura homogeneizada sobre SLF4J/Logback para estandarizar las trazas de auditoría técnica, asegurando que los identificadores de correlación y seguimiento viajen a través de los hilos de ejecución de los microservicios.
* **DTO Utils:** Factorías de conversión automática de entidades, utilidades de mapeo profundo (*deep copy*) y utilidades de saneamiento para prevenir la persistencia de datos corruptos o ataques de inyección básica.