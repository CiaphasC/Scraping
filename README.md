
# **Curso Completo: Procesamiento de Datos desde Streams con Clean Architecture en C#**

## **Índice del Curso**

1. **Introducción: ¿Qué aprenderás en este curso?**
2. **Conceptos Fundamentales: Streams en C#**
   - ¿Qué es un stream?
   - Operaciones básicas con streams.
3. **Introducción a Clean Architecture**
   - ¿Qué es Clean Architecture?
   - Principios SOLID en la arquitectura.
4. **Estructura del Proyecto: Análisis del Código**
   - Dominio (Core)
   - Aplicación (Application)
   - Infraestructura (Infrastructure)
   - Presentación (Presentation)
5. **Desglose Detallado del Código**
   - Solicitud HTTP y manejo del stream.
   - Procesamiento de datos con `Utf8JsonReader`.
6. **Procesamiento Incremental y Flujo de Control**
   - Flujo de ejecución explicado paso a paso.
   - Uso de delegados y desacoplamiento.
7. **Práctica Avanzada: Optimización y Mejoras**
   - Procesamiento paralelo y pipelines.
   - Manejo de errores y logging.
8. **Conclusiones y Siguientes Pasos**
9. **Optimización de Rendimiento**
   - Reducción del tamaño del buffer.
   - Uso de `MemoryPool<byte>`.
   - Procesamiento por lotes.
   - Balanceo dinámico de consumidores.
10. **Pruebas Automatizadas**
    - Pruebas unitarias e integración.
    - Mocking de servicios.
11. **Buenas Prácticas y Documentación**
    - Uso de dependencias y logging.
    - Monitoreo y métricas.
    - Documentación y preparación del código.

---

## **Introducción: ¿Qué aprenderás en este curso?**

En este curso aprenderás a manejar grandes volúmenes de datos de manera eficiente utilizando streams en C#. Aprenderás a:
- **Procesar datos en tiempo real** sin cargarlos completamente en memoria.
- **Aplicar principios de Clean Architecture** para crear sistemas escalables.
- **Optimizar el rendimiento** de tu aplicación utilizando técnicas avanzadas como procesamiento paralelo y pipelines.
- **Manejar errores** de manera eficiente y registrar eventos usando **ILogger**.

---


# Módulo 2: Streams en C#

### ¿Qué es un Stream?

Un **Stream** en C# es una secuencia de bytes que fluye desde un origen hasta un destino. Los Streams se utilizan para leer o escribir datos de manera **secuencial** y **gradual**. La principal ventaja de usar Streams es que no es necesario cargar todo el archivo o datos en memoria a la vez, lo que es crucial cuando se maneja una gran cantidad de datos o archivos grandes.

#### Tipos de Streams en C#:
1. **FileStream**: Para trabajar con archivos en disco.
2. **MemoryStream**: Utiliza la memoria para almacenar los datos.
3. **NetworkStream**: Para leer y escribir datos desde una conexión de red.
4. **BufferedStream**: Agrega un búfer de memoria para optimizar las operaciones de lectura y escritura.

---

### Lectura Asíncrona desde un Stream

Uno de los aspectos más poderosos de los Streams es su capacidad para leer datos de manera asíncrona. Esto significa que podemos leer fragmentos de datos sin bloquear el hilo principal de ejecución, lo cual es muy útil en aplicaciones de alto rendimiento.

#### Código 1: Lectura Asíncrona de un Stream

```csharp
private async Task ProcessStreamAsync(Stream stream, Action<ExchangeRateDetail> processEntry)
{
    var buffer = new byte[8192];  // Creamos un buffer de 8192 bytes
    int bytesRead;  // Variable para almacenar la cantidad de bytes leídos

    // Mientras haya datos por leer en el stream
    while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
    {
        var reader = new Utf8JsonReader(new ReadOnlySpan<byte>(buffer, 0, bytesRead));

        // Procesar cada fragmento de datos leídos
        while (reader.Read())
        {
            if (reader.TokenType == JsonTokenType.StartObject)
            {
                var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
                if (entry != null)
                    processEntry(entry);  // Procesamos la entrada (callback)
            }
        }
    }
}
```

### Desglosando el Código

1. **Buffer de Lectura**:
   ```csharp
   var buffer = new byte[8192];  // 8 KB de tamaño para el buffer
   ```
   Se crea un **buffer** de 8192 bytes (8 KB) para almacenar los datos que se leen del **Stream**.

2. **Lectura Asíncrona**:
   ```csharp
   bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
   ```
   Aquí, estamos usando `ReadAsync` para leer los datos del **Stream** de manera asíncrona. `await` asegura que el hilo principal no se bloquee mientras se leen los datos.

3. **Procesamiento de Datos JSON**:
   ```csharp
   var reader = new Utf8JsonReader(new ReadOnlySpan<byte>(buffer, 0, bytesRead));
   ```
   Usamos `Utf8JsonReader` para leer los datos de tipo **JSON** dentro del buffer.

4. **Lectura de Objetos JSON**:
   ```csharp
   while (reader.Read())
   {
       if (reader.TokenType == JsonTokenType.StartObject)
       {
           var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
           if (entry != null)
               processEntry(entry);
       }
   }
   ```
   Este ciclo lee cada token dentro del JSON y deserializa el objeto cuando encuentra un token de tipo `StartObject`.

---

### Escritura Asíncrona en un Stream

Al igual que leemos de un Stream, podemos escribir datos en un **Stream** de manera asíncrona. Esto también ayuda a evitar bloqueos en el hilo principal.

#### Código 2: Escritura Asíncrona en un Stream

```csharp
using var stream = new FileStream("archivo.txt", FileMode.Create);  // Abrir archivo para escritura
byte[] buffer = Encoding.UTF8.GetBytes("Hola, Mundo");  // Convertir el texto a bytes
await stream.WriteAsync(buffer, 0, buffer.Length);  // Escribir en el stream de forma asíncrona
```

---

### Beneficios de Usar Streams

1. **Eficiencia en Memoria**: Los Streams permiten procesar fragmentos de datos sin cargarlos completamente en memoria.
2. **Procesamiento Incremental**: Los Streams permiten procesar datos conforme llegan.
3. **Optimización de Desempeño**: Al leer o escribir de manera asíncrona, el uso de Streams optimiza el tiempo de respuesta de la aplicación.

---

### Diagrama de Procesamiento Asíncrono con Streams

```plaintext
[Start] -> [Initialize Stream] -> [ReadAsync Data] -> [Process Data in Buffer] 
     |                      |                        |
     v                      v                        v
 [Data Processed] -> [WriteAsync Data] -> [Close Stream]
```

---

### Conclusión del Módulo 2

En este módulo, aprendiste cómo funcionan los **Streams** en C# para leer y escribir datos de manera eficiente y asíncrona, mejorando el rendimiento en aplicaciones que procesan grandes volúmenes de datos. Los **Streams** son herramientas fundamentales cuando se trabaja con archivos grandes, redes o cualquier tipo de datos que necesite ser procesado sin sobrecargar la memoria.

---

**Siguientes Pasos:**
- Experimenta con la lectura y escritura de Streams en tus propios proyectos.
- Profundiza en el uso de otros tipos de **Streams** como `MemoryStream` y `NetworkStream`.
- Explora la **serialización de datos JSON** y cómo se puede usar `Utf8JsonReader` para procesar datos complejos de manera eficiente.



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


## **Capítulo 4: Análisis del Código**

### **Estructura del Proyecto**

```plaintext
ExchangeRateApp/
├── Core/
│   ├── ExchangeRateDetail.cs
│   ├── ExchangeRateResult.cs
│   ├── IExchangeRateApi.cs
│   └── IExchangeRateProcessor.cs
├── Application/
│   └── ExchangeRateProcessor.cs
├── Infrastructure/
│   └── ExchangeRateApi.cs
└── Presentation/
    └── Program.cs
```

---

## **Capítulo 5: Desglose Detallado del Código**

### **Solicitud HTTP y Manejo del Stream**
La aplicación realiza una solicitud HTTP usando `HttpClient` y recibe un stream de respuesta. Este stream es procesado en fragmentos para evitar cargar toda la respuesta en memoria.

```csharp
await using var responseStream = await response.Content.ReadAsStreamAsync();
await ProcessStreamAsync(responseStream, processEntry);
```

### **Procesamiento de Datos con `Utf8JsonReader`**
Usamos `Utf8JsonReader` para procesar los datos sin cargar el JSON completo en memoria.

```csharp
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
```

---

## **Capítulo 6: Procesamiento Incremental y Flujo de Control**

El flujo de ejecución sigue estos pasos:
1. **Entrada del usuario**: El programa solicita el año y el mes al usuario.
2. **Construcción del Payload**: El programa crea un payload para la solicitud a la API.
3. **Solicitud HTTP**: El payload se envía y se recibe el stream.
4. **Procesamiento**: A medida que se reciben fragmentos, se procesan y se almacenan.

---

## **Capítulo 7: Práctica Avanzada - Optimización y Mejoras**

### **Procesamiento Paralelo**
En lugar de procesar los datos secuencialmente, puedes usar varios hilos (productores y consumidores) para manejar múltiples tareas simultáneamente.

#### Código de Productor y Consumidor
```csharp
var producer = Task.Run(async () =>
{
    while ((bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length)) > 0)
    {
        bufferQueue.Add(buffer);
    }
});

var consumers = Enumerable.Range(0, Environment.ProcessorCount).Select(_ => Task.Run(() =>
{
    foreach (var fragment in bufferQueue.GetConsumingEnumerable())
    {
        // Procesar fragmento...
    }
})).ToArray();

await Task.WhenAll(producer, Task.WhenAll(consumers));
```

---

## **Capítulo 8: Manejo de Errores y Logging**

### **Uso de ILogger**
Registra todos los errores y eventos importantes usando `ILogger` para depurar y monitorear el sistema.

```csharp
_logger.LogError($"Error en el productor: {ex.Message}");
```

---

## **Capítulo 9: Optimización de Rendimiento**

### **Reducción del Tamaño del Buffer**
El tamaño del buffer puede ajustarse para mejorar la eficiencia. Si se tienen grandes volúmenes de datos, un buffer más grande puede ser más rápido.

```csharp
var buffer = new byte[32768]; // 32 KB
```

### **Uso de `MemoryPool<byte>`**
Usar `MemoryPool<byte>` ayuda a reducir la asignación y fragmentación de memoria, mejorando la eficiencia.

---

## **Capítulo 10: Pruebas Automatizadas**

### **Pruebas Unitarias**
Las pruebas unitarias aseguran que el código funcione correctamente en cualquier circunstancia. Aquí tienes un ejemplo para probar el procesador de datos:

```csharp
[Fact]
public void ProcessEntry_ShouldStoreDataCorrectly()
{
    var processor = new ExchangeRateProcessor();
    processor.ProcessEntry(new ExchangeRateDetail { Date = "2025-01-01", Value = "3.75", Type = "C" });
    Assert.Single(processor.GetResults());
}
```

---

## **Capítulo 11: Buenas Prácticas y Documentación**

### **Uso de Dependencias**
Usa inyección de dependencias para los componentes como `HttpClient` y `ILogger`:

```csharp
services.AddHttpClient<ExchangeRateApi>();
services.AddLogging();
```

### **Monitoreo y Métricas**
Implementa monitoreo para medir el rendimiento del sistema y detectar cuellos de botella.

---

## **Conclusión**

Con este curso, ahora sabes cómo:
1. Procesar datos desde un stream de manera eficiente.
2. Diseñar y organizar tu código con **Clean Architecture**.
3. Optimizar el rendimiento y manejar errores correctamente.

### **¿Estás listo para aplicar lo aprendido?**

¡Gracias por participar en este curso! 😊


