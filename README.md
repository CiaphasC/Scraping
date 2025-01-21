Curso Completo: Procesamiento de Datos desde Streams con Clean Architecture en C#
Índice del Curso
Introducción: ¿Qué aprenderás en este curso?
Conceptos Fundamentales: Streams en C#
¿Qué es un stream?
Operaciones básicas con streams.
Introducción a Clean Architecture
¿Qué es Clean Architecture?
Principios SOLID en la arquitectura.
Estructura del Proyecto: Análisis del Código
Dominio (Core)
Aplicación (Application)
Infraestructura (Infrastructure)
Presentación (Presentation)
Desglose Detallado del Código
Solicitud HTTP y manejo del stream.
Procesamiento de datos con Utf8JsonReader.
Procesamiento Incremental y Flujo de Control
Flujo de ejecución explicado paso a paso.
Uso de delegados y desacoplamiento.
Práctica Avanzada: Optimización y Mejoras
Procesamiento paralelo y pipelines.
Manejo de errores y logging.
Conclusiones y Siguientes Pasos
Optimización de Rendimiento
Reducción del tamaño del buffer.
Uso de MemoryPool<byte>.
Procesamiento por lotes.
Balanceo dinámico de consumidores.
Pruebas Automatizadas
Pruebas unitarias e integración.
Mocking de servicios.
Buenas Prácticas y Documentación
Uso de dependencias y logging.
Monitoreo y métricas.
Documentación y preparación del código.
Capítulo 1: Introducción
¿Qué aprenderás en este curso?
En este curso, aprenderás cómo manejar grandes volúmenes de datos de manera eficiente utilizando streams en C#. Aprenderás a:

Procesar datos en tiempo real sin cargarlos completamente en memoria.
Aplicar principios de Clean Architecture para crear sistemas escalables.
Optimizar el rendimiento de tu aplicación utilizando técnicas avanzadas como procesamiento paralelo y pipelines.
Manejar errores de manera eficiente y registrar eventos usando ILogger.
Capítulo 2: Streams en C#
¿Qué es un Stream?
Un stream es una secuencia de datos que fluye de un origen a un destino. Es un flujo continuo de bytes que se puede leer o escribir de manera incremental. Los streams son una parte esencial para manejar grandes cantidades de datos sin sobrecargar la memoria del sistema.

Operaciones Básicas con Streams
En C#, los streams pueden ser utilizados para leer y escribir datos de manera eficiente. Aquí tienes algunos ejemplos clave:

Ejemplo de Lectura desde un Stream
csharp
Copiar
using System;
using System.IO;

class StreamExample
{
    static async Task Main()
    {
        using var stream = new FileStream("archivo.txt", FileMode.Open);
        byte[] buffer = new byte[1024];
        int bytesRead = await stream.ReadAsync(buffer, 0, buffer.Length);
        Console.WriteLine($"Bytes leídos: {bytesRead}");
    }
}
Ejemplo de Escritura en un Stream
csharp
Copiar
using System;
using System.IO;

class StreamExample
{
    static async Task Main()
    {
        using var stream = new FileStream("output.txt", FileMode.Create);
        byte[] data = new byte[] { 65, 66, 67 }; // "ABC"
        await stream.WriteAsync(data, 0, data.Length);
        Console.WriteLine("Datos escritos.");
    }
}
Capítulo 3: Clean Architecture
¿Qué es Clean Architecture?
Clean Architecture es un patrón de diseño que organiza el código en capas para asegurar que cada parte del sistema tenga una única responsabilidad y que las dependencias fluyan en una dirección predecible.

Capas de Clean Architecture
Dominio (Core): Contiene las reglas de negocio y entidades centrales.
Aplicación (Application): Implementa los casos de uso y lógica de aplicación.
Infraestructura (Infrastructure): Maneja conexiones externas, como APIs o bases de datos.
Presentación (Presentation): Controla la interacción con el usuario.
Principios SOLID
S: Responsabilidad única (Single Responsibility).
O: Abierto/Cerrado (Open/Closed).
L: Sustitución de Liskov (Liskov Substitution).
I: Segregación de Interfaces (Interface Segregation).
D: Inversión de Dependencias (Dependency Inversion).
Capítulo 4: Análisis del Código
Estructura del Proyecto
plaintext
Copiar
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
Core: Dominio
Clase ExchangeRateDetail
Representa un registro JSON obtenido de la API:

json
Copiar
{
  "fecPublica": "2025-01-01",
  "valTipo": "3.75",
  "codTipo": "C"
}
Código:

csharp
Copiar
public class ExchangeRateDetail
{
    [JsonPropertyName("fecPublica")]
    public string Date { get; set; } = string.Empty;

    [JsonPropertyName("valTipo")]
    public string Value { get; set; } = string.Empty;

    [JsonPropertyName("codTipo")]
    public string Type { get; set; } = string.Empty;
}
Clase ExchangeRateResult
Convierte los datos JSON en un formato amigable:

csharp
Copiar
public class ExchangeRateResult
{
    public string Fecha { get; set; } = string.Empty;
    public string Compra { get; set; } = "N/A";
    public string Venta { get; set; } = "N/A";
}
Application: Procesamiento
Clase ExchangeRateProcessor
csharp
Copiar
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
Infrastructure: Conexión con la API
Clase ExchangeRateApi
csharp
Copiar
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
            mes = month - 1,
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
Capítulo 5: Procesamiento Incremental y Flujo de Control
Flujo de Ejecución Explicado
Entrada del usuario: El programa solicita el año y el mes al usuario.
Construcción del payload: El programa crea el payload con los datos del año y mes.
Solicitud HTTP: El payload se envía a la API para obtener los datos de tipo de cambio.
Procesamiento de datos: A medida que los datos llegan, son procesados en fragmentos.
Capítulo 6: Práctica Avanzada - Optimización y Mejoras
Procesamiento Paralelo
El procesamiento paralelo mejora el rendimiento al permitir que múltiples hilos procesen los datos al mismo tiempo. Usamos un Productor para descargar los datos y Consumidores para procesarlos en paralelo.

Código de Productor y Consumidor
csharp
Copiar
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
Capítulo 7: Manejo de Errores y Logging
Uso de Logging
Usar ILogger para registrar errores y eventos es una buena práctica en sistemas paralelos, ya que te permite saber qué está sucediendo en cada parte del proceso.

Ejemplo de Logging
csharp
Copiar
_logger.LogError($"Error en el productor: {ex.Message}");
Capítulo 8: Optimización de Rendimiento
1. Reducción del Tamaño del Buffer
El tamaño del buffer afecta la cantidad de memoria usada y el rendimiento. Ajusta el tamaño del buffer dependiendo de las características de tu sistema.

csharp
Copiar
var buffer = new byte[32768]; // 32 KB
2. Uso de MemoryPool<byte>
Usar MemoryPool<byte> reduce la asignación y fragmentación de memoria, mejorando la eficiencia.

Capítulo 9: Pruebas Automatizadas
1. Pruebas Unitarias
Usa pruebas unitarias para garantizar que los componentes clave como ExchangeRateProcessor funcionen correctamente.

Ejemplo de Prueba Unitaria
csharp
Copiar
[Fact]
public void ProcessEntry_ShouldStoreDataCorrectly()
{
    var processor = new ExchangeRateProcessor();
    processor.ProcessEntry(new ExchangeRateDetail { Date = "2025-01-01", Value = "3.75", Type = "C" });
    Assert.Single(processor.GetResults());
}
Capítulo 10: Buenas Prácticas y Documentación
1. Uso de Dependencias
Usa inyección de dependencias para configurar componentes como HttpClient y ILogger.

csharp
Copiar
services.AddHttpClient<ExchangeRateApi>();
services.AddLogging();
2. Monitoreo y Métricas
Implementa monitoreo para medir el rendimiento del sistema y detectar cuellos de botella.

Conclusión
Este curso te ha proporcionado una comprensión completa de cómo manejar y procesar grandes volúmenes de datos de manera eficiente usando streams en C#, siguiendo los principios de Clean Architecture y utilizando técnicas avanzadas como el procesamiento paralelo.

Ahora estás listo para construir aplicaciones escalables y eficientes que procesen datos en tiempo real sin comprometer la memoria.

