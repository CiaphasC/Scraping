
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

## **Capítulo 2: Streams en C#**

### **¿Qué es un stream?**
Un stream es una secuencia de datos que fluye de un origen a un destino. Es un flujo continuo de bytes que se puede leer o escribir de manera incremental. Los streams son una parte esencial para manejar grandes cantidades de datos sin sobrecargar la memoria del sistema.

### **Operaciones Básicas con Streams**

#### **Ejemplo de Lectura desde un Stream**
```csharp
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
```

#### **Ejemplo de Escritura en un Stream**
```csharp
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
```

---

## **Capítulo 3: Clean Architecture**

### **¿Qué es Clean Architecture?**

Clean Architecture es un enfoque de diseño de software que busca separar la lógica de negocio de los detalles de implementación. Esto garantiza que las aplicaciones sean modulares, fáciles de probar y de mantener.

### **Capas de Clean Architecture**
1. **Dominio (Core)**: Contiene las reglas de negocio y las entidades principales.
2. **Aplicación (Application)**: Implementa los casos de uso.
3. **Infraestructura (Infrastructure)**: Maneja las dependencias externas como APIs, bases de datos, etc.
4. **Presentación (Presentation)**: Controla la interacción con el usuario.

#### **Principios SOLID**
- **S**: Responsabilidad única (Single Responsibility).
- **O**: Abierto/Cerrado (Open/Closed).
- **L**: Sustitución de Liskov (Liskov Substitution).
- **I**: Segregación de Interfaces (Interface Segregation).
- **D**: Inversión de Dependencias (Dependency Inversion).

---

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


