
# **Curso Completo: Procesamiento de Datos desde Streams con Clean Architecture en C#**




# Módulo 1: Introducción al Curso y Fundamentos del Proyecto

Este es el primer módulo de un curso detallado en el que aprenderemos a **trabajar con Streams en C#** y a aplicar la **Clean Architecture** en proyectos de software. A lo largo de este curso, abordaremos los conceptos fundamentales de cómo organizar el código, optimizar el rendimiento y garantizar que las aplicaciones sean escalables y fáciles de mantener.

En este módulo, proporcionamos una **introducción general** a los conceptos y las herramientas que utilizaremos en el curso. Nos centraremos en el análisis del código base, que se utiliza para obtener tasas de cambio desde una API externa, procesar los datos de manera eficiente y presentarlos al usuario.

## Objetivos del Módulo

1. **Entender la estructura básica de un proyecto en C#** que sigue los principios de **Clean Architecture**.
2. **Aprender a consumir y procesar datos desde una API externa**.
3. **Introducir los conceptos de Streams en C#** y su uso para manejar grandes volúmenes de datos de manera eficiente.
4. **Implementar el procesamiento de datos de forma asíncrona** para optimizar el rendimiento de la aplicación.

Este módulo es clave para establecer las bases de cómo estructurar y organizar el código en aplicaciones modernas y escalables.

---

## 1. ¿Qué es Clean Architecture?

**Clean Architecture** es un patrón de diseño que promueve la **separación de responsabilidades** dentro de una aplicación, lo que hace que el código sea más fácil de entender, probar y mantener. La idea principal es separar la lógica de negocio del código relacionado con la infraestructura, como el acceso a bases de datos, servicios externos o la interfaz de usuario.

### Estructura de Clean Architecture

La arquitectura limpia divide el código en capas bien definidas. Cada capa tiene una responsabilidad distinta y solo puede depender de capas **más internas**. Las capas son:

1. **Capa de Dominio (Core)**: Contiene las entidades del negocio y las reglas que gobiernan el sistema. Esta capa es **independiente** de cualquier infraestructura o tecnología.
2. **Capa de Aplicación (Application)**: Implementa la lógica de negocio o casos de uso de la aplicación. Esta capa interactúa con la capa de **Dominio** y la de **Infraestructura**.
3. **Capa de Infraestructura (Infrastructure)**: Esta capa maneja los detalles técnicos, como el acceso a bases de datos, servicios web, API externas, etc.
4. **Capa de Presentación (Presentation)**: Se encarga de la interfaz con el usuario (por ejemplo, aplicaciones de consola, aplicaciones web, etc.).

El siguiente diagrama ilustra cómo se estructuran las capas en Clean Architecture utilizando Mermaid:

```mermaid
graph TD
    A[Presentation Layer] --> B[Application Layer]
    B --> C[Domain Layer]
    C --> D[Infrastructure Layer]
```

---

## 2. Fundamentos del Código Base

El código que vamos a analizar y trabajar en este curso está diseñado para consumir datos de una API pública que ofrece las tasas de cambio de divisas. A continuación, detallaremos cómo cada parte del código contribuye a la estructura del proyecto.

### **Clase `ExchangeRateDetail` (Capa de Dominio)**

```csharp
public class ExchangeRateDetail
{
    [JsonPropertyName("fecPublica")]
    public string Date { get; set; } = string.Empty;

    [JsonPropertyName("valTipo")]
    public string Value { get; set; } = string.Empty;

    [JsonPropertyName("codTipo")]
    public string Type { get; set; } = string.Empty;
}
```

**Explicación:**
- Esta clase **representa una entidad de dominio** que contiene los detalles de las tasas de cambio. Está definida en la capa de **Dominio**, que es donde residen las reglas del negocio.
- Los atributos como `Date`, `Value` y `Type` corresponden a los datos que obtendremos desde la API.
- Usamos **`JsonPropertyName`** para mapear las propiedades JSON a las propiedades del objeto C#.

### **Clase `ExchangeRateProcessor` (Capa de Aplicación)**

```csharp
public class ExchangeRateProcessor : IExchangeRateProcessor
{
    private readonly Dictionary<string, ExchangeRateResult> _results = new();

    public void ProcessEntry(ExchangeRateDetail entry)
    {
        if (!_results.ContainsKey(entry.Date))
            _results[entry.Date] = new ExchangeRateResult { Fecha = entry.Date };

        if (entry.Type == "C")
            _results[entry.Date].Compra = entry.Value;
        else if (entry.Type == "V")
            _results[entry.Date].Venta = entry.Value;
    }

    public IEnumerable<ExchangeRateResult> GetResults() => _results.Values;
}
```

**Explicación:**
- La clase `ExchangeRateProcessor` es parte de la **capa de aplicación** y se encarga de procesar los datos recibidos desde la API.
- **`ProcessEntry`** toma un objeto `ExchangeRateDetail` y lo procesa. Si la fecha no existe en el diccionario `_results`, se crea una nueva entrada.
- Dependiendo del tipo (`C` para compra, `V` para venta), se asigna el valor correspondiente.
- Finalmente, **`GetResults`** devuelve los resultados procesados.

### **Clase `ExchangeRateApi` (Capa de Infraestructura)**

```csharp
public class ExchangeRateApi : IExchangeRateApi
{
    private const string ApiUrl = "https://e-consulta.sunat.gob.pe/cl-at-ittipcam/tcS01Alias/listarTipoCambio";
    private const string ApiToken = "xon89wgbu8o6330q1mida90siv1mtkj1";

    private readonly HttpClient _httpClient;

    public ExchangeRateApi(HttpClient httpClient)
    {
        _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
    }

    public async Task ProcessExchangeRatesAsync(int year, int month, Action<ExchangeRateDetail> processEntry)
    {
        var payload = new
        {
            anio = year,
            mes = month - 1, // Ajuste de mes para la API
            token = ApiToken
        };

        using var payloadStream = new MemoryStream();
        await JsonSerializer.SerializeAsync(payloadStream, payload);
        payloadStream.Position = 0;

        using var content = new StreamContent(payloadStream);
        content.Headers.ContentType = new System.Net.Http.Headers.MediaTypeHeaderValue("application/json");

        using var response = await _httpClient.PostAsync(ApiUrl, content);
        response.EnsureSuccessStatusCode();

        await using var responseStream = await response.Content.ReadAsStreamAsync();
        await ProcessStreamAsync(responseStream, processEntry);
    }

    private async Task ProcessStreamAsync(Stream stream, Action<ExchangeRateDetail> processEntry)
    {
        var buffer = new byte[8192];
        int bytesRead;

        while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
        {
            var reader = new Utf8JsonReader(new ReadOnlySpan<byte>(buffer, 0, bytesRead));

            while (reader.Read())
            {
                if (reader.TokenType == JsonTokenType.StartObject)
                {
                    var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
                    if (entry != null)
                        processEntry(entry);
                }
            }
        }
    }
}
```

**Explicación:**
- La clase `ExchangeRateApi` es parte de la **capa de infraestructura** y se encarga de interactuar con la API externa.
- Se usa `HttpClient` para hacer una solicitud **POST** a la API, enviando los parámetros requeridos (año, mes y token).
- La respuesta de la API se recibe en formato JSON, que luego se procesa en **fragmentos** utilizando **Streams** asíncronos.

---

## 3. Procesamiento de Datos Asíncronos

### **¿Por qué usar procesamiento asíncrono?**

En aplicaciones que interactúan con servicios externos, como APIs, el uso de **operaciones asíncronas** es crucial para no bloquear el hilo principal de ejecución. Cuando solicitamos datos desde una API externa, la operación puede tardar algún tiempo. Si no utilizamos asincronía, la aplicación se detendría hasta que obtuviéramos la respuesta, lo que afectaría la capacidad de respuesta de la interfaz de usuario y otros procesos.

---

## Diagrama del Flujo del Proyecto con Mermaid

El siguiente diagrama muestra cómo se procesan los datos a través de las diferentes capas de la aplicación, siguiendo la **Clean Architecture**:

```mermaid
graph TD
    A[Presentation Layer] --> B[Application Layer]
    B --> C[Domain Layer]
    C --> D[Infrastructure Layer]
```

Este diagrama ilustra cómo las capas interactúan, con **Presentación** (interfaz de usuario) en la parte superior, pasando a la **Aplicación** (casos de uso), luego a la **Dominio** (reglas de negocio), y finalmente a la **Infraestructura** (servicios externos como APIs).

---

## Conclusión del Módulo 1

En este módulo, aprendiste sobre la **estructura básica de una aplicación en C#** siguiendo la **Clean Architecture**, y cómo se implementan los principios de separación de responsabilidades en un proyecto real. A través del ejemplo de consumo de datos desde una API externa, entendiste cómo procesar datos de manera asíncrona utilizando **Streams**.

El uso de **Clean Architecture** permite que tu código sea más modular, escalable y fácil de mantener. La interacción con **APIs externas** y el **procesamiento eficiente de datos** son componentes clave en muchas aplicaciones modernas.

---

**Siguientes Pasos:**
- Implementa **Clean Architecture** en proyectos más complejos.
- Profundiza en el uso de **Streams** y **operaciones asíncronas** en C#.
- Experimenta con el consumo de otras **APIs** y el procesamiento de datos en tiempo real.





# Módulo 2: Streams en C# - Procesamiento de Datos Asíncrono

En este módulo, vamos a profundizar en el uso de **Streams en C#** y cómo procesar datos de manera **eficiente y asíncrona**. Cuando trabajamos con grandes volúmenes de datos, ya sea desde una API externa, archivos locales o incluso en flujos de datos en tiempo real, es esencial utilizar un enfoque que optimice el rendimiento y reduzca el uso de recursos, como la memoria.

A lo largo de este módulo, veremos cómo usar **Streams asíncronos** para manejar grandes cantidades de datos sin que la aplicación se vea bloqueada. Los diagramas **Mermaid** se utilizarán para ilustrar cada concepto y mostrar cómo fluye el procesamiento de datos desde su lectura hasta su presentación al usuario.

## ¿Qué es un Stream en C#?

Un **Stream** en C# es una secuencia de datos que se transmite de manera continua desde un **origen** a un **destino**. Los Streams son una de las herramientas más poderosas cuando se trata de leer o escribir datos de manera eficiente sin cargar todo el contenido en memoria a la vez, lo que es esencial cuando se manejan grandes volúmenes de datos.

### Tipos de Streams en C#

1. **FileStream**: Usado para leer y escribir archivos locales en disco.
2. **MemoryStream**: Utiliza la memoria del sistema como almacenamiento temporal de los datos.
3. **NetworkStream**: Facilita la lectura y escritura de datos sobre una red.
4. **BufferedStream**: Añade un búfer de memoria para mejorar el rendimiento de la lectura y escritura de datos.

## Ventajas del uso de Streams en el procesamiento de datos

- **Eficiencia en la Memoria**: Los Streams permiten procesar datos grandes sin necesidad de cargar todo el contenido en memoria.
- **Rendimiento Mejorado**: Al leer y escribir en fragmentos, se optimiza el uso de la CPU y la memoria.
- **Procesamiento Asíncrono**: Los Streams asíncronos permiten que el programa siga funcionando mientras lee o escribe los datos, evitando bloqueos innecesarios.

---

## Diagrama del Flujo de Trabajo con Streams

El flujo de procesamiento de datos en un **Stream asíncrono** sigue un patrón en el que los datos se leen en fragmentos, se procesan, y se pueden escribir nuevamente o mostrar al usuario. Aquí hay un diagrama **Mermaid** que ilustra este flujo.

```mermaid
graph LR
    A[Inicio] --> B[Inicialización del Stream]
    B --> C[Leer Datos Asíncronos en Fragmentos]
    C --> D[Procesar Datos Leídos]
    D --> E[Escribir Datos Asíncronos o Mostrar Resultados]
    E --> F[Cerrar el Stream]
    F --> G[Fin]
```

### **Explicación del Diagrama:**

1. **Inicialización del Stream**: En el inicio, configuramos el **Stream** (por ejemplo, un `FileStream`, `NetworkStream`, etc.) para comenzar a leer o escribir los datos.
2. **Lectura Asíncrona en Fragmentos**: Usamos la lectura asíncrona (`ReadAsync`) para obtener los datos en fragmentos de tamaño fijo. Esto permite que el programa no se bloquee mientras espera los datos.
3. **Procesamiento de los Datos**: Una vez que los datos son leídos en fragmentos, se procesan según el tipo de aplicación (en nuestro caso, deserializando JSON).
4. **Escritura Asíncrona o Presentación de Datos**: Los datos procesados pueden ser escritos en otro archivo o mostrados al usuario. Esto se realiza de manera asíncrona para no bloquear el hilo principal.
5. **Cierre del Stream**: Finalmente, se cierra el **Stream** para liberar los recursos.

---

## Procesamiento Asíncrono en C#

### **¿Qué es el Procesamiento Asíncrono?**

El procesamiento asíncrono en **C#** permite que las operaciones de **entrada/salida** (como leer de un archivo o hacer una solicitud HTTP) se realicen sin bloquear el hilo principal de la aplicación. Esto es crucial cuando interactuamos con APIs externas o manejamos grandes archivos, ya que el programa puede seguir ejecutando otras tareas mientras espera a que se completen las operaciones de E/S.

### **Beneficios del Procesamiento Asíncrono**:

1. **Evita Bloqueos del Hilo Principal**: Mientras se realizan operaciones de E/S, el hilo principal sigue disponible para realizar otras tareas.
2. **Mejora la Capacidad de Respuesta**: Las aplicaciones que usan asincronía responden más rápido y son más interactivas, ya que no dependen de que se complete una operación antes de continuar.
3. **Optimización de los Recursos del Sistema**: Al permitir que múltiples operaciones se realicen al mismo tiempo, se aprovechan mejor los recursos del sistema, como la CPU y la memoria.

### **Código de Lectura Asíncrona desde un Stream**

En este ejemplo, mostramos cómo leer de manera asíncrona los datos de una API externa que devuelve las tasas de cambio:

```csharp
private async Task ProcessStreamAsync(Stream stream, Action<ExchangeRateDetail> processEntry)
{
    var buffer = new byte[8192];  // Buffer de 8192 bytes
    int bytesRead;

    // Leemos los datos de manera asíncrona en fragmentos
    while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
    {
        var reader = new Utf8JsonReader(new ReadOnlySpan<byte>(buffer, 0, bytesRead));

        // Procesamos cada fragmento de datos leídos
        while (reader.Read())
        {
            if (reader.TokenType == JsonTokenType.StartObject)
            {
                var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
                if (entry != null)
                    processEntry(entry);  // Procesamos los datos
            }
        }
    }
}
```

### **Desglosando el Código:**

1. **Buffer de Lectura**:
   El buffer es un arreglo de bytes que almacenará temporalmente los datos que leemos del **Stream**. Al leer los datos en fragmentos pequeños, podemos manejar grandes volúmenes de datos sin consumir demasiada memoria.

2. **Lectura Asíncrona**:
   El uso de `await stream.ReadAsync` permite que la operación de lectura no bloquee el hilo principal de la aplicación, lo que mejora la capacidad de respuesta del sistema.

3. **Procesamiento de Datos**:
   Una vez que los datos son leídos en fragmentos, los procesamos utilizando un **JsonReader** para interpretar los datos JSON.

4. **Callback para Procesar los Datos**:
   Los datos deserializados son enviados al callback `processEntry`, que puede almacenarlos, procesarlos o mostrarlos.

---

## Diagrama de Flujo del Procesamiento de Datos en Streams Asíncronos

El siguiente diagrama **Mermaid** describe cómo fluye el procesamiento de datos de manera asíncrona y sin bloquear el hilo principal.

```mermaid
graph TD
    A[Inicio] --> B[Iniciar Lectura Asíncrona]
    B --> C[Leer Datos en Fragmentos]
    C --> D[Deserializar y Procesar los Datos]
    D --> E[Escribir Datos Asíncronos o Mostrar Resultados]
    E --> F[Cerrar Stream]
    F --> G[Fin]
```

### **Explicación del Diagrama:**

1. **Iniciar Lectura Asíncrona**: Al comenzar el proceso, se establece la conexión con el **Stream** (por ejemplo, un archivo o una conexión de red).
2. **Leer Datos en Fragmentos**: Se leen los datos en **fragmentos** utilizando operaciones asíncronas, para asegurar que no se bloquee el hilo principal.
3. **Deserializar y Procesar los Datos**: Los datos en formato JSON se deserializan y se procesan para ser almacenados o utilizados.
4. **Escribir Datos o Mostrar Resultados**: Los datos procesados se pueden escribir en un archivo o mostrar al usuario de manera asíncrona.
5. **Cerrar el Stream**: Finalmente, se cierra el **Stream** para liberar los recursos.

---

## Comparación con Otros Lenguajes

A continuación, compararemos cómo se maneja el procesamiento de **Streams** en **C#**, **Node.js** (JavaScript) y **Python**:

#### **Node.js (JavaScript)**

En **Node.js**, el procesamiento de **Streams** se maneja usando el módulo `fs` para leer archivos y flujos de datos de manera asíncrona:

```javascript
const fs = require('fs');
const stream = fs.createReadStream('archivo.txt');
stream.on('data', (chunk) => {
  console.log(chunk.toString());
});
```

**Diferencias**:
- **C#** usa `ReadAsync` para leer de manera asíncrona, mientras que en **Node.js** se usan eventos.
- **C#** es un lenguaje con **tipado estático**, lo que ofrece mayor seguridad en tiempo de compilación, mientras que **Node.js** es más flexible debido a su **tipado dinámico**.

#### **Python**

En **Python**, el procesamiento de archivos se realiza de manera síncrona por defecto:

```python
with open('archivo.txt', 'r') as file:
    for line in file:
        print(line)
```

**Diferencias**:
- En **C#**, se utilizan **Streams asíncronos** para evitar bloqueos, mientras que en **Python**, las operaciones de lectura de archivos son **bloqueantes** por defecto.

---

## Conclusión del Módulo 2

En este módulo, exploramos cómo usar **Streams en C#** para manejar grandes volúmenes de datos de manera eficiente y asíncrona. Aprendimos sobre los beneficios de **la asincronía** y cómo aplicar este concepto para evitar bloqueos innecesarios en la ejecución de la aplicación.

La capacidad de leer y escribir datos de manera asíncrona permite mejorar el rendimiento y optimizar el uso de recursos en aplicaciones que interactúan con APIs, bases de datos o archivos grandes.

---

**Siguientes Pasos:**
- Experimenta con **Streams asíncronos** en proyectos más complejos.
- Profundiza en el uso de **MemoryStream** y **NetworkStream** para otras aplicaciones.
- Aprende más sobre el uso de **Serialización JSON** y cómo interactuar con APIs externas.




# Módulo 3: Clean Architecture en C#

En este módulo, aprenderemos en detalle qué es la **Clean Architecture**, cómo aplicarla en C# y cómo ayuda a construir aplicaciones escalables, mantenibles y fácilmente testeables. También discutiremos cómo esta arquitectura se diferencia de otras arquitecturas comunes y cómo se implementa en C#.

La **Clean Architecture** es un patrón de diseño de software que promueve la separación de responsabilidades entre las diferentes capas de la aplicación, de modo que cada una de ellas se especializa en un conjunto de tareas bien definido. Este enfoque permite que el código sea más modular, fácil de probar y mantener, además de hacer que la aplicación sea más fácil de extender sin afectar el resto del sistema.

## ¿Qué es la Clean Architecture?

La **Clean Architecture** fue propuesta por **Robert C. Martin** (también conocido como **Uncle Bob**) y es una forma de estructurar el software para que sea independiente de las herramientas y frameworks que se usan. Esto se logra creando una **división clara** entre el núcleo de la aplicación (la lógica de negocio) y el resto de la infraestructura (como bases de datos, interfaces de usuario, etc.).

### Principios Fundamentales de la Clean Architecture

1. **Independencia de Frameworks**: Los frameworks y herramientas deben ser intercambiables sin afectar la lógica de negocio.
2. **Independencia de la Interfaz de Usuario**: La interfaz de usuario (UI) puede cambiar sin que afecte el resto de la aplicación.
3. **Independencia de la Base de Datos**: La base de datos y la lógica de acceso a datos pueden cambiar sin que afecten la lógica de negocio.
4. **Testabilidad**: El sistema debe ser fácil de probar de forma aislada.

### Principios SOLID

La Clean Architecture hace un uso extensivo de los principios **SOLID**, que son principios fundamentales para el desarrollo de software orientado a objetos. Vamos a ver cómo se aplican estos principios en la arquitectura.

---

## Estructura de la Clean Architecture

La Clean Architecture se organiza en **capas** que se comunican de forma específica. Las capas más externas dependen de las capas más internas, pero no al revés.

Las capas son las siguientes:

1. **Capa de Dominio (Core)**: Esta capa contiene las **entidades** (lógica de negocio) y las reglas del negocio que son independientes de cualquier infraestructura externa.
2. **Capa de Aplicación (Application)**: Contiene los casos de uso de la aplicación, orquestando las interacciones entre las capas externas e internas.
3. **Capa de Infraestructura (Infrastructure)**: Contiene detalles como la interacción con la base de datos, servicios web, API externas, etc.
4. **Capa de Presentación (Presentation)**: La interfaz de usuario o cualquier interacción con el usuario.

### Diagramas de la Clean Architecture

El siguiente diagrama ilustra cómo se estructuran las capas de la Clean Architecture:

```plaintext
+----------------------------------+
|          Presentation Layer      |
|   (Interfaces de Usuario)       |
+----------------------------------+
              |
              v
+----------------------------------+
|        Application Layer        |
|   (Casos de uso, lógica)        |
+----------------------------------+
              |
              v
+----------------------------------+
|           Domain Layer           |
|  (Entidades y reglas del negocio)|
+----------------------------------+
              |
              v
+----------------------------------+
|      Infrastructure Layer       |
|   (Acceso a bases de datos,    |
|    APIs externas, etc.)        |
+----------------------------------+
```

---

## Aplicación de la Clean Architecture en C#

### **Capa de Dominio (Core)**

La capa de **Dominio** contiene las entidades, que son los objetos de negocio principales, y las reglas que rigen la lógica de negocio.

#### **Código: Entidad ExchangeRateDetail**

```csharp
public class ExchangeRateDetail
{
    [JsonPropertyName("fecPublica")]
    public string Date { get; set; } = string.Empty;

    [JsonPropertyName("valTipo")]
    public string Value { get; set; } = string.Empty;

    [JsonPropertyName("codTipo")]
    public string Type { get; set; } = string.Empty;
}
```

**Explicación:**
- La clase `ExchangeRateDetail` es una **entidad de dominio** que representa una tasa de cambio. Contiene propiedades como `Date`, `Value` y `Type`, que son fundamentales para la lógica de negocio.
- La entidad **no depende de ninguna infraestructura externa** como bases de datos o interfaces de usuario.

#### **Principios SOLID Aplicados en la Capa de Dominio:**
- **S (Responsabilidad Única)**: La clase `ExchangeRateDetail` tiene una única responsabilidad: representar un detalle de tasa de cambio.
- **O (Abierto/Cerrado)**: Puedes extender la clase `ExchangeRateDetail` si es necesario, pero no necesitas modificarla.
- **L (Sustitución de Liskov)**: Si una clase hereda de `ExchangeRateDetail`, debe poder ser utilizada en lugar de esta sin causar problemas en el sistema.

---

### **Capa de Aplicación (Application)**

La capa de **Aplicación** contiene los casos de uso de la aplicación. Los casos de uso orquestan las interacciones entre la lógica de negocio (capa de Dominio) y la infraestructura (bases de datos, servicios externos, etc.).

#### **Código: Caso de Uso - ExchangeRateProcessor**

```csharp
public class ExchangeRateProcessor : IExchangeRateProcessor
{
    private readonly Dictionary<string, ExchangeRateResult> _results = new();

    public void ProcessEntry(ExchangeRateDetail entry)
    {
        if (!_results.ContainsKey(entry.Date))
            _results[entry.Date] = new ExchangeRateResult { Fecha = entry.Date };

        if (entry.Type == "C")
            _results[entry.Date].Compra = entry.Value;
        else if (entry.Type == "V")
            _results[entry.Date].Venta = entry.Value;
    }

    public IEnumerable<ExchangeRateResult> GetResults() => _results.Values;
}
```

**Explicación:**
- La clase `ExchangeRateProcessor` maneja la lógica de **procesar** las tasas de cambio y almacenar los resultados. Esta es la **capa de aplicación** que contiene los **casos de uso**.
- **Orquesta** el flujo de datos entre la capa de **Dominio** y la capa de **Infraestructura**.

---

### **Capa de Infraestructura (Infrastructure)**

La capa de **Infraestructura** maneja los detalles de la implementación de los servicios externos, como el acceso a bases de datos o la interacción con APIs externas. Esta capa debe depender de las **interfaces** definidas en las capas superiores, pero nunca debe contener lógica de negocio.

#### **Código: Interacción con API - ExchangeRateApi**

```csharp
public class ExchangeRateApi : IExchangeRateApi
{
    private const string ApiUrl = "https://api.exchangeratesapi.io/latest";
    private readonly HttpClient _httpClient;

    public ExchangeRateApi(HttpClient httpClient)
    {
        _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
    }

    public async Task ProcessExchangeRatesAsync(int year, int month, Action<ExchangeRateDetail> processEntry)
    {
        var response = await _httpClient.GetStringAsync($"{ApiUrl}?year={year}&month={month}");
        var rates = JsonSerializer.Deserialize<List<ExchangeRateDetail>>(response);
        foreach (var rate in rates)
        {
            processEntry(rate);
        }
    }
}
```

**Explicación:**
- La clase `ExchangeRateApi` se encarga de interactuar con una API externa para obtener las tasas de cambio.
- Utiliza el **HttpClient** para hacer una solicitud HTTP a la API y luego procesa la respuesta.

---

### **Capa de Presentación (Presentation)**

La capa de **Presentación** es donde interactuamos con el usuario o con cualquier sistema de interfaz. En este caso, la presentación podría ser una aplicación de consola que muestra las tasas de cambio procesadas.

#### **Código: Interfaz de Usuario (Console)**

```csharp
class Program
{
    static async Task Main(string[] args)
    {
        var httpClient = new HttpClient();
        var api = new ExchangeRateApi(httpClient);
        var processor = new ExchangeRateProcessor();

        await api.ProcessExchangeRatesAsync(2025, 1, processor.ProcessEntry);

        foreach (var result in processor.GetResults())
        {
            Console.WriteLine($"Fecha: {result.Fecha} | Compra: {result.Compra} | Venta: {result.Venta}");
        }
    }
}
```

**Explicación:**
- El código de **Presentación** es una simple aplicación de consola que solicita las tasas de cambio y las muestra al usuario.
- Aquí, la lógica de negocio ya ha sido procesada en las capas inferiores, y la **Presentación** solo se encarga de mostrar los resultados.

---

### Diferencias con Otras Arquitecturas

1. **Monolítica**: En una **arquitectura monolítica**, todo el código (interfaz de usuario, lógica de negocio, acceso a datos) reside en un único bloque. Si bien más sencillo al principio, esto puede volverse difícil de mantener a medida que el sistema crece.

2. **MVC**: Aunque el patrón **MVC (Modelo-Vista-Controlador)** también organiza el código en capas, la **Clean Architecture** va un paso más allá, separando la lógica de negocio del acceso a datos y la interfaz de usuario, lo que facilita la escalabilidad y el mantenimiento.

3. **Microservicios**: En comparación con **microservicios**, la **Clean Architecture** se enfoca más en la **organización interna** del código dentro de un único servicio, mientras que los microservicios abogan por la creación de servicios independientes con su propia base de datos.

---

## Conclusión del Módulo

En este módulo, aprendiste sobre la **Clean Architecture** y cómo aplicarla en C#. Esta arquitectura promueve la separación de responsabilidades y permite construir aplicaciones modulares, escalables y fáciles de mantener.

Al seguir los principios de **SOLID** y separar las distintas responsabilidades en capas, puedes crear software que sea independiente de frameworks, bases de datos, interfaces de usuario, y que sea fácilmente testeable.

---

**Siguientes Pasos:**
- Implementa Clean Architecture en tus propios proyectos.
- Investiga más sobre cómo integrar Clean Architecture con tecnologías como **ASP.NET Core**, **Entity Framework**, y **Web APIs**.



# Módulo 4: Consumo de la API y Procesamiento de Datos en JSON

En este módulo, aprenderemos cómo consumir datos de una **API externa** utilizando C#, cómo deserializar los datos JSON que recibimos y cómo procesarlos de manera eficiente en el contexto de la **Clean Architecture**. Este módulo es fundamental para trabajar con APIs RESTful y para integrar datos externos en aplicaciones de software.

El consumo de APIs es una parte importante del desarrollo de software moderno, ya que muchas aplicaciones dependen de servicios externos para obtener datos, ya sea desde bases de datos remotas, otras aplicaciones o servicios en la nube.

## Consumo de APIs en C#

En C#, podemos consumir una API externa utilizando la clase `HttpClient`. Este es el cliente HTTP utilizado para hacer solicitudes HTTP, como GET, POST, PUT, DELETE, etc., a una API. `HttpClient` es una de las herramientas más poderosas cuando trabajamos con servicios web y APIs RESTful.

### 1. **Realizando Solicitudes HTTP en C# con HttpClient**

#### **Código de Solicitud GET:**
```csharp
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        using var client = new HttpClient();
        var response = await client.GetStringAsync("https://api.exchangeratesapi.io/latest");
        Console.WriteLine(response);
    }
}
```

**Explicación:**
- **`HttpClient`**: Es la clase principal para realizar solicitudes HTTP en C#. En el código, creamos una nueva instancia de `HttpClient` utilizando la palabra clave `using`, lo que garantiza que se liberen los recursos una vez que ya no se necesiten.
- **`GetStringAsync`**: Se utiliza para realizar una solicitud **GET** de manera asíncrona. Esta solicitud obtiene el contenido de la API y lo devuelve como una cadena de texto.

**Diagrama de flujo:**

```plaintext
[Start] -> [Create HttpClient] -> [GET Request] -> [Receive Response] -> [Print Response]
```

---

### 2. **Procesamiento de Datos JSON en C#**

Cuando trabajamos con APIs RESTful, la mayoría de las respuestas se devuelven en formato **JSON**. En C#, podemos procesar JSON de manera eficiente utilizando `JsonSerializer` para **deserializar** y **serializar** datos entre objetos C# y JSON.

#### **Código de Deserialización JSON:**
```csharp
using System;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        using var client = new HttpClient();
        var response = await client.GetStringAsync("https://api.exchangeratesapi.io/latest");

        // Deserializar el JSON a un objeto C#
        var exchangeRates = JsonSerializer.Deserialize<ExchangeRates>(response);
        
        // Mostrar los datos deserializados
        Console.WriteLine($"Base: {exchangeRates.Base}");
        foreach (var rate in exchangeRates.Rates)
        {
            Console.WriteLine($"{rate.Key}: {rate.Value}");
        }
    }
}

public class ExchangeRates
{
    public string Base { get; set; }
    public Dictionary<string, decimal> Rates { get; set; }
}
```

**Explicación:**
- **`GetStringAsync`**: Como antes, realiza la solicitud GET de manera asíncrona.
- **`JsonSerializer.Deserialize<T>`**: Se utiliza para deserializar la respuesta JSON en un objeto C#. En este caso, deserializamos la respuesta JSON en un objeto `ExchangeRates`.
- **`Dictionary<string, decimal>`**: En este ejemplo, usamos un diccionario para almacenar las tasas de cambio (valores asociados a cada tipo de moneda).

#### **Diagrama de flujo:**

```plaintext
[Start] -> [GET Request to API] -> [Get JSON Response] -> [Deserialize JSON to Object] -> [Display Data]
```

---

### 3. **Manejo de Errores en el Consumo de la API**

Es importante manejar errores correctamente cuando consumimos APIs. Esto incluye gestionar las respuestas con códigos de estado diferentes a 200 (OK), como 404 (Not Found) o 500 (Internal Server Error).

#### **Código de Manejo de Errores:**
```csharp
using System;
using System.Net.Http;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        using var client = new HttpClient();
        try
        {
            var response = await client.GetAsync("https://api.exchangeratesapi.io/latest");
            response.EnsureSuccessStatusCode();  // Lanza una excepción si la respuesta no es exitosa

            var responseBody = await response.Content.ReadAsStringAsync();
            Console.WriteLine(responseBody);
        }
        catch (HttpRequestException e)
        {
            Console.WriteLine($"Error de solicitud: {e.Message}");
        }
    }
}
```

**Explicación:**
- **`EnsureSuccessStatusCode`**: Este método lanza una excepción si el código de estado HTTP de la respuesta no es exitoso (es decir, no es un 2xx).
- **`catch (HttpRequestException e)`**: Captura cualquier excepción que ocurra durante la solicitud HTTP, como problemas de red o una respuesta con un error HTTP.

---

### 4. **Integración del Consumo de la API en la Clean Architecture**

El consumo de APIs debe integrarse adecuadamente en la **Clean Architecture**, donde la **capa de infraestructura** es responsable de interactuar con servicios externos, como APIs.

#### **Código de Interfaz de API (Capa de Infraestructura)**:
```csharp
public interface IExchangeRateApi
{
    Task<ExchangeRates> GetExchangeRatesAsync();
}
```

**Explicación:**
- La interfaz `IExchangeRateApi` define el contrato para el consumo de la API. Esto asegura que la implementación de la API esté desacoplada de otras partes de la aplicación.

#### **Código de Implementación de API (Capa de Infraestructura)**:
```csharp
public class ExchangeRateApi : IExchangeRateApi
{
    private const string ApiUrl = "https://api.exchangeratesapi.io/latest";
    private readonly HttpClient _httpClient;

    public ExchangeRateApi(HttpClient httpClient)
    {
        _httpClient = httpClient ?? throw new ArgumentNullException(nameof(httpClient));
    }

    public async Task<ExchangeRates> GetExchangeRatesAsync()
    {
        var response = await _httpClient.GetStringAsync(ApiUrl);
        return JsonSerializer.Deserialize<ExchangeRates>(response);
    }
}
```

**Explicación:**
- La clase `ExchangeRateApi` implementa la interfaz `IExchangeRateApi` y es responsable de hacer la solicitud HTTP a la API, procesar la respuesta y devolver los datos deserializados.

#### **Uso en la Capa de Aplicación**:
```csharp
public class ExchangeRateService
{
    private readonly IExchangeRateApi _exchangeRateApi;

    public ExchangeRateService(IExchangeRateApi exchangeRateApi)
    {
        _exchangeRateApi = exchangeRateApi;
    }

    public async Task DisplayExchangeRates()
    {
        var rates = await _exchangeRateApi.GetExchangeRatesAsync();
        foreach (var rate in rates.Rates)
        {
            Console.WriteLine($"{rate.Key}: {rate.Value}");
        }
    }
}
```

**Explicación:**
- El servicio `ExchangeRateService` es parte de la **capa de aplicación** y utiliza la interfaz `IExchangeRateApi` para obtener las tasas de cambio a través de la capa de infraestructura.

---

### Diferencias en Otros Lenguajes de Programación

Aunque el proceso de consumir una API y procesar JSON es similar en muchos lenguajes de programación, la implementación y las herramientas utilizadas varían. A continuación se presentan algunas diferencias clave entre **C#** y otros lenguajes:

#### **JavaScript (Node.js)**

En **JavaScript**, generalmente se usa `fetch` o `axios` para realizar solicitudes HTTP, y `JSON.parse()` para deserializar los datos:

```javascript
fetch('https://api.exchangeratesapi.io/latest')
  .then(response => response.json())
  .then(data => console.log(data));
```

**Diferencias**:
- **C#** usa `HttpClient` y `JsonSerializer`, mientras que en **JavaScript** se usan `fetch` y `JSON.parse()`.
- **C#** tiene un sistema de tipos estáticos, mientras que **JavaScript** es dinámico.

#### **Python**

En **Python**, el módulo `requests` se usa comúnmente para hacer solicitudes HTTP, y `json` para procesar JSON:

```python
import requests
response = requests.get('https://api.exchangeratesapi.io/latest')
data = response.json()
print(data)
```

**Diferencias**:
- **C#** es más estructurado debido a su tipado estático, mientras que **Python** es más dinámico y directo.

---

## Conclusión del Módulo 4

En este módulo, aprendimos cómo consumir APIs RESTful utilizando **C#**, cómo procesar los datos **JSON** que recibimos de esas APIs y cómo integrar este proceso en el contexto de la **Clean Architecture**. Al utilizar la clase `HttpClient` para realizar solicitudes HTTP, junto con `JsonSerializer` para deserializar JSON, podemos manejar de manera eficiente los datos provenientes de servicios externos.

El consumo de APIs es una parte fundamental en el desarrollo de aplicaciones modernas, y ahora tienes las herramientas necesarias para integrar de manera efectiva servicios externos en tus aplicaciones.

---

**Siguientes Pasos:**
- Experimenta con la integración de APIs en tus propios proyectos.
- Profundiza en el manejo de datos JSON y otras fuentes de datos externas.
- Explora la **serialización y deserialización avanzada** en C# y cómo trabajar con datos complejos.


# Módulo 5: Optimización del Rendimiento con Procesamiento Asíncrono

En este módulo, aprenderemos cómo optimizar el rendimiento de nuestras aplicaciones utilizando **procesamiento asíncrono** en C#. El objetivo principal es mejorar la capacidad de respuesta y la eficiencia en aplicaciones que manejan grandes volúmenes de datos o realizan múltiples tareas simultáneamente, como en el caso del código que consume datos de una API externa.

### ¿Por qué es importante la optimización del rendimiento?

Las aplicaciones modernas, especialmente las que interactúan con servicios externos como APIs, deben ser capaces de procesar grandes volúmenes de datos de manera eficiente. Cuando no se optimiza el rendimiento, la aplicación puede volverse lenta, innecesariamente intensiva en recursos, y puede bloquear el hilo principal de ejecución, lo que provoca una mala experiencia de usuario.

### **El procesamiento asíncrono** es una de las herramientas más poderosas para optimizar el rendimiento. Utilizando técnicas como `async` y `await`, podemos evitar que el hilo principal se bloquee mientras esperamos que se completen operaciones de entrada/salida (E/S) o cálculos largos.

---

## Procesamiento Asíncrono en C#

### ¿Qué es el procesamiento asíncrono?

El procesamiento **asíncrono** en C# permite realizar tareas sin bloquear el hilo principal de la aplicación. En lugar de esperar a que una operación termine antes de continuar con el flujo de ejecución, el procesamiento asíncrono permite que otras tareas se ejecuten mientras se esperan los resultados.

En aplicaciones que necesitan interactuar con APIs, acceder a archivos grandes, o realizar cálculos complejos, el procesamiento asíncrono es crucial para mantener la fluidez y la eficiencia del sistema.

### ¿Cómo mejora el rendimiento?

1. **No bloquea el hilo principal**: Las operaciones de E/S (como leer un archivo o consultar una API) no bloquean el hilo principal de la aplicación.
2. **Mejora la capacidad de respuesta**: La aplicación sigue ejecutando otras tareas mientras espera que se completen las operaciones asíncronas.
3. **Aprovecha al máximo los recursos**: Utiliza el **multi-threading** de manera eficiente, permitiendo que el sistema ejecute múltiples tareas al mismo tiempo sin bloquear el flujo de trabajo principal.

---

### **Código de Optimización del Rendimiento con Streams Asíncronos**

En el siguiente código, se utiliza un **Stream asíncrono** para procesar los datos de una API sin bloquear el hilo principal. Este ejemplo es una parte crucial de cómo optimizar el rendimiento en una aplicación que interactúa con un servicio externo.

#### **Código: Lectura Asíncrona de la API con Streams**

```csharp
private async Task ProcessStreamAsync(Stream stream, Action<ExchangeRateDetail> processEntry)
{
    var buffer = new byte[8192];  // Buffer de 8192 bytes
    int bytesRead;

    // Leer el Stream en fragmentos sin bloquear el hilo principal
    while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
    {
        var reader = new Utf8JsonReader(new ReadOnlySpan<byte>(buffer, 0, bytesRead));

        // Procesar los datos de cada fragmento leído
        while (reader.Read())
        {
            if (reader.TokenType == JsonTokenType.StartObject)
            {
                var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
                if (entry != null)
                    processEntry(entry);  // Procesamos los datos
            }
        }
    }
}
```

### Desglosando el Código:

1. **Buffer de Lectura**:
   ```csharp
   var buffer = new byte[8192];  // Buffer de 8 KB
   ```
   Creamos un **buffer** de 8192 bytes, que es un tamaño adecuado para leer fragmentos grandes de datos de manera eficiente sin sobrecargar la memoria.

2. **Lectura Asíncrona**:
   ```csharp
   bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
   ```
   Usamos `ReadAsync` para leer los datos de manera asíncrona. Esto significa que el hilo principal **no se bloquea** mientras los datos son leídos. El procesamiento puede continuar con otras tareas, como actualizaciones de la interfaz de usuario o cálculos adicionales, mientras se espera por los datos.

3. **Deserialización de JSON**:
   ```csharp
   var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
   ```
   Una vez que leemos un fragmento de datos, los deserializamos en un objeto `ExchangeRateDetail` que se puede utilizar dentro de la lógica de la aplicación. Utilizando **`Utf8JsonReader`**, procesamos los datos en formato JSON.

4. **Callback para Procesar los Datos**:
   ```csharp
   processEntry(entry);
   ```
   El callback `processEntry` se utiliza para manejar los datos procesados, ya sea almacenándolos en memoria o mostrándolos en la interfaz de usuario.

---

### Optimización con **Paralelismo y Concurrente** en Streams

Para obtener un **rendimiento aún mayor**, podemos usar paralelismo en la lectura y procesamiento de datos. En vez de procesar los datos de forma secuencial, podemos dividir el trabajo en **tareas paralelas** que se ejecutan al mismo tiempo.

#### **Código: Lectura Concurrente en Paralelo**

```csharp
private async Task ProcessStreamsInParallel(Stream stream, Action<ExchangeRateDetail> processEntry)
{
    var buffer = new byte[8192];
    int bytesRead;

    var tasks = new List<Task>();

    while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
    {
        tasks.Add(Task.Run(() => 
        {
            var reader = new Utf8JsonReader(new ReadOnlySpan<byte>(buffer, 0, bytesRead));
            while (reader.Read())
            {
                if (reader.TokenType == JsonTokenType.StartObject)
                {
                    var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
                    if (entry != null)
                        processEntry(entry);
                }
            }
        }));
    }

    await Task.WhenAll(tasks);  // Esperamos que todas las tareas terminen
}
```

**Explicación**:
- **`Task.Run`** se utiliza para ejecutar cada fragmento de lectura y procesamiento en un hilo paralelo, lo que permite realizar múltiples operaciones de manera concurrente.
- **`Task.WhenAll`** espera a que todas las tareas paralelas terminen, lo que asegura que el procesamiento completo se haya completado.

---

### **Beneficios de la Optimización Asíncrona en C#**

1. **No Bloqueo del Hilo Principal**:
   La principal ventaja del procesamiento asíncrono es que las operaciones de E/S, como las solicitudes HTTP o la lectura de archivos, no bloquean el hilo principal. Esto mejora la capacidad de respuesta de la aplicación.

2. **Mejora del Rendimiento**:
   Utilizando el procesamiento en paralelo, podemos manejar múltiples fragmentos de datos al mismo tiempo, maximizando el uso de los recursos disponibles y acelerando el procesamiento.

3. **Optimización de Recursos**:
   Al realizar operaciones de E/S de manera asíncrona, evitamos el desperdicio de recursos que ocurre cuando el sistema está esperando a que se complete una operación de bloque. Esto resulta en aplicaciones más eficientes.

---

### Diagrama de Flujo del Procesamiento Asíncrono con Paralelismo

El siguiente diagrama ilustra cómo el procesamiento asíncrono y paralelo puede mejorar el rendimiento de la aplicación al manejar tareas simultáneamente:

```plaintext
[Start] -> [Initialize Stream] -> [ReadAsync Data in Chunks] 
     |                        |                   |
     v                        v                   v
[Process Data in Parallel] -> [Run Tasks Concurrently] -> [Task Completion]
     |                        |                   |
     v                        v                   v
[WriteAsync Data] -> [End]
```

---

### Comparación con Otras Técnicas de Optimización

#### **Optimización Síncrona vs. Asíncrona**

En el procesamiento **síncrono**, cada operación espera a que la anterior termine antes de continuar. Esto puede resultar en un rendimiento deficiente, especialmente cuando se realizan múltiples operaciones de E/S. En cambio, el procesamiento **asíncrono** permite que la aplicación siga ejecutándose mientras espera que se completen operaciones de E/S.

#### **Optimización con Caching**

Otra forma de optimizar el rendimiento es usando **caching**. En lugar de hacer solicitudes a la API cada vez que se necesita información, podemos almacenar los resultados en caché para evitar hacer solicitudes redundantes.

---

## Conclusión del Módulo 5

En este módulo, aprendiste cómo utilizar **procesamiento asíncrono** y **paralelismo** en C# para optimizar el rendimiento de una aplicación que interactúa con APIs o maneja grandes volúmenes de datos. El uso de **Streams asíncronos** y técnicas como `Task.WhenAll` y `Task.Run` permite mejorar la eficiencia de la aplicación sin comprometer su capacidad de respuesta.

---

**Siguientes Pasos:**
- Experimenta con el procesamiento asíncrono en proyectos que interactúan con servicios externos.
- Investiga técnicas avanzadas de **paralelismo** y **concurrencia** en C# para maximizar el rendimiento en aplicaciones complejas.
- Profundiza en el uso de **caching** y otras optimizaciones de rendimiento en sistemas distribuidos.
