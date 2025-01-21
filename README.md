
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


