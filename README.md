
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



# Módulo 2: Streams en C# - Procesamiento de Datos Asíncrono

En este módulo, aprenderemos cómo trabajar con **Streams en C#** para leer y escribir datos de manera eficiente, especialmente en situaciones donde manejamos grandes volúmenes de información. A través de un ejemplo práctico, entenderemos cómo los **Streams** y la **asincronía** en C# pueden optimizar el rendimiento de una aplicación que consume datos de una API.

Vamos a analizar el siguiente código, que consume tasas de cambio desde una API, procesa los datos en **fragmentos** usando **Streams** y los presenta al usuario.

## ¿Qué es un Stream?

Un **Stream** en C# es una secuencia de bytes que fluye desde un origen hasta un destino. Los Streams se utilizan para leer o escribir datos de manera **secuencial** y **gradual**. La principal ventaja de usar Streams es que no es necesario cargar todo el archivo o datos en memoria a la vez, lo que es crucial cuando se maneja una gran cantidad de datos o archivos grandes.

### Tipos de Streams en C#

1. **FileStream**: Para trabajar con archivos en disco.
2. **MemoryStream**: Utiliza la memoria para almacenar los datos.
3. **NetworkStream**: Para leer y escribir datos desde una conexión de red.
4. **BufferedStream**: Agrega un búfer de memoria para optimizar las operaciones de lectura y escritura.

---

## Lectura Asíncrona desde un Stream

Uno de los aspectos más poderosos de los Streams es su capacidad para leer datos de manera asíncrona. Esto significa que podemos leer fragmentos de datos sin bloquear el hilo principal de ejecución, lo cual es muy útil en aplicaciones de alto rendimiento.

La lectura asíncrona permite que la aplicación no se bloquee mientras espera que los datos lleguen. Esto es ideal cuando trabajamos con APIs o grandes archivos.

### **Código: Lectura Asíncrona de un Stream**

En este código, vamos a procesar las respuestas de la API **Sunat** (que devuelve las tasas de cambio) de manera asíncrona, usando un **Stream**:

```csharp
private async Task ProcessStreamAsync(Stream stream, Action<ExchangeRateDetail> processEntry)
{
    var buffer = new byte[8192];  // Buffer de 8192 bytes
    int bytesRead;

    // Leemos el Stream en fragmentos de 8192 bytes
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
                    processEntry(entry);  // Procesamos la entrada
            }
        }
    }
}
```

### **Desglosando el Código**

1. **Buffer de Lectura**:
   ```csharp
   var buffer = new byte[8192];  // 8 KB de tamaño para el buffer
   ```
   El buffer es un arreglo de bytes que utilizamos para almacenar temporalmente los datos leídos del Stream. Esto nos permite leer fragmentos pequeños en lugar de cargar todo el contenido en memoria a la vez.

2. **Lectura Asíncrona**:
   ```csharp
   bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
   ```
   `ReadAsync` es un método asíncrono que lee los datos del Stream. Utilizamos `await` para asegurarnos de que la operación no bloquee el hilo principal de ejecución, permitiendo que el programa continúe haciendo otras tareas mientras los datos se leen.

3. **Deserialización de JSON**:
   ```csharp
   var entry = JsonSerializer.Deserialize<ExchangeRateDetail>(ref reader);
   ```
   Usamos `JsonSerializer.Deserialize` para convertir el fragmento de JSON leído en un objeto de tipo `ExchangeRateDetail`. Este paso es crucial porque los datos de la API vienen en formato JSON y necesitamos convertirlos a un tipo que podamos manipular en C#.

4. **Callback para Procesar los Datos**:
   ```csharp
   processEntry(entry);
   ```
   Después de deserializar los datos, pasamos el objeto `ExchangeRateDetail` al callback `processEntry`, que se encargará de almacenarlos o procesarlos de alguna manera.

---

### Escritura Asíncrona en un Stream

Al igual que leemos datos desde un Stream, también podemos escribir datos en un Stream de manera asíncrona. Esta operación es útil cuando, por ejemplo, queremos guardar los resultados procesados de vuelta en un archivo o en un servicio externo.

### **Código: Escritura Asíncrona en un Stream**

```csharp
using var stream = new FileStream("archivo.txt", FileMode.Create);  // Abrir archivo para escritura
byte[] buffer = Encoding.UTF8.GetBytes("Hola, Mundo");  // Convertir el texto a bytes
await stream.WriteAsync(buffer, 0, buffer.Length);  // Escribir en el stream de forma asíncrona
```

**Desglosando el Código:**
1. **Abrir un Archivo para Escritura**:
   `FileStream` se usa para abrir un archivo en el sistema de archivos. El modo `FileMode.Create` indica que se creará el archivo si no existe o se sobrescribirá si ya existe.

2. **Convertir Texto a Bytes**:
   Convertimos una cadena de texto a un arreglo de bytes utilizando `Encoding.UTF8.GetBytes`.

3. **Escribir Asíncronamente**:
   `WriteAsync` permite escribir en el archivo de manera asíncrona, lo que no bloquea el hilo principal de ejecución mientras se realiza la operación.

---

### **Beneficios de Usar Streams en C#**

1. **Eficiencia en Memoria**:
   Los Streams permiten manejar grandes volúmenes de datos sin cargar todo el archivo o datos en memoria a la vez. Esto es fundamental para aplicaciones que deben procesar archivos grandes o recibir datos en tiempo real.

2. **Optimización de Desempeño**:
   Al procesar los datos en fragmentos, los Streams permiten que la aplicación sea más eficiente, ya que evita la sobrecarga de cargar grandes bloques de datos en memoria.

3. **Procesamiento Incremental**:
   Al leer y escribir datos en fragmentos pequeños, podemos procesar los datos a medida que llegan, lo que mejora el rendimiento de las aplicaciones que interactúan con APIs, bases de datos o sistemas de almacenamiento distribuidos.

---

### Diagrama de Procesamiento Asíncrono con Streams

Aquí te muestro un diagrama de flujo que describe cómo funciona el procesamiento de datos en un Stream asíncrono:

```plaintext
[Start] -> [Initialize Stream] -> [ReadAsync Data] -> [Process Data in Buffer] 
     |                      |                        |
     v                      v                        v
 [Data Processed] -> [WriteAsync Data] -> [Close Stream]
```

- **Iniciar Stream**: Abrimos el Stream, ya sea para leer o escribir.
- **Leer de manera asíncrona**: Leemos los datos en fragmentos sin bloquear el hilo principal.
- **Procesar los datos**: Deserializamos o procesamos los datos leídos.
- **Escribir de manera asíncrona**: Si es necesario, escribimos los resultados procesados.
- **Cerrar el Stream**: Cerramos el Stream para liberar los recursos.

---

### Comparación con Otros Lenguajes

Aunque el concepto de Streams es común en muchos lenguajes, su implementación varía. A continuación se muestra una breve comparación de cómo se implementa el procesamiento de Streams en otros lenguajes.

#### **JavaScript (Node.js)**

En **Node.js**, se usan flujos (`Streams`) y el módulo `fs` (file system) para leer y escribir archivos de manera asíncrona:

```javascript
const fs = require('fs');

const stream = fs.createReadStream('archivo.txt');
stream.on('data', chunk => {
  console.log(chunk.toString());
});
```

**Diferencias**:
- En **C#**, usamos `Stream.ReadAsync` para leer datos de manera asíncrona, mientras que en **Node.js**, usamos eventos para manejar los datos a medida que llegan.
- **C#** ofrece un sistema de tipos estáticos que mejora la seguridad en tiempo de compilación, mientras que **Node.js** es más flexible debido a su tipado dinámico.

#### **Python**

En **Python**, podemos usar el módulo `io` para manejar Streams:

```python
with open('archivo.txt', 'r') as file:
    for line in file:
        print(line)
```

**Diferencias**:
- **C#** utiliza el patrón de **Streams asíncronos** para no bloquear el hilo principal, mientras que en **Python** la lectura de archivos es más directa y síncrona por defecto.

---

## Conclusión del Módulo 2

En este módulo, aprendiste cómo funcionan los **Streams** en C#, cómo usarlos para leer y escribir datos de manera eficiente y asíncrona, y cómo aprovechar estas herramientas para procesar grandes volúmenes de datos. El uso de **Streams** es esencial en aplicaciones que necesitan manejar archivos grandes, datos en tiempo real o cuando interactúan con servicios externos.

Los conceptos de **asincronía** y **procesamiento incremental** proporcionan una manera eficiente de gestionar datos sin bloquear el hilo principal, mejorando el rendimiento general de la aplicación.

---

**Siguientes Pasos:**
- Experimenta con **Streams asíncronos** en proyectos más complejos.
- Explora el uso de **MemoryStream** y **NetworkStream** para otras aplicaciones.
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


