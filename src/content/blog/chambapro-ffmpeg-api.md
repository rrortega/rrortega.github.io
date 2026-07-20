---
title: 'Convertir audio con Rust y FFmpeg en milisegundos'
description: 'Crea una API ultra rápida en Rust con FFmpeg para optimizar transcripciones de OpenAI Whisper en integraciones de mensajería.'
pubDate: 'Jul 19 2026'
heroImage: 'https://opengraph.githubassets.com/8ec9919eeb7baeab4090eaed242cd15033af0fb105f48a60e996f75f63930a63/rrortega/chambapro-ffmpeg-api'
---

Integras notas de voz de WhatsApp en tu agente de AI, pero la API de Whisper tarda una eternidad o falla constantemente. Ya sea que uses agregadores como ManyChat, yCloud, Kaypso, o la API oficial de Meta para WhatsApp Business, los audios llegan en formato `.oga` u `.ogg`, los cuales ralentizan el Speech-to-Text. Whisper procesa la información muchísimo más rápido si le entregas un archivo `.mp3` limpio y optimizado a 128kbps.

## ¿Por qué esto importa hoy?

Las interfaces de voz dominan los agentes conversacionales modernos. Cada segundo de retraso en la transcripción destruye la retención del usuario final. 

Si procesas conversiones de audio pesadas de forma síncrona o sobrecargas tu servidor web, la latencia se dispara. Necesitas un microservicio dedicado que intercepte el audio original, lo convierta en milisegundos y mantenga el consumo de RAM en cero.

## Creando un convertidor desde cero

Rust es la herramienta ideal para esta tarea por su seguridad y control de memoria. Utilizaremos el framework Axum para levantar un servidor HTTP asíncrono y controlaremos FFmpeg mediante subprocesos no bloqueantes.

A continuación, construimos esta solución paso a paso. Diseñamos un flujo de conversión que procesa las solicitudes sin comprometer el rendimiento general del servidor.

### Paso 1: Configurar el entorno de Rust

Iniciamos un nuevo proyecto binario usando Cargo en la terminal.

```bash
cargo new ffmpeg-api-post
cd ffmpeg-api-post
```

Salida esperada de la terminal:
```text
     Created binary (application) `ffmpeg-api-post` package
```

Ahora, configuramos las dependencias necesarias en el archivo `Cargo.toml`.

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1.35", features = ["full"] }
tokio-util = { version = "0.7", features = ["io"] }
uuid = { version = "1.6", features = ["v4"] }
```

### Paso 2: Servidor base y recepción de archivos

Configuramos el enrutador HTTP de Axum para que reciba archivos multimedia mediante peticiones multipart en el endpoint `/convert`.

```rust
use axum::{
    extract::Multipart,
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::post,
    Router,
};
use tokio::fs::File;
use tokio::io::AsyncWriteExt;
use uuid::Uuid;

async fn convert_handler(mut multipart: Multipart) -> Result<Response, StatusCode> {
    let mut temp_input_path = None;
    let uuid = Uuid::new_v4().to_string();

    while let Some(field) = multipart.next_field().await.map_err(|_| StatusCode::BAD_REQUEST)? {
        if field.name() == Some("file") {
            let path = format!("/tmp/upload_{}.oga", uuid);
            let mut file = File::create(&path).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
            let data = field.bytes().await.map_err(|_| StatusCode::BAD_REQUEST)?;
            file.write_all(&data).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
            temp_input_path = Some(path);
        }
    }

    let input_path = temp_input_path.ok_or(StatusCode::BAD_REQUEST)?;
    let output_path = format!("/tmp/converted_{}.mp3", uuid);
    
    // Aquí ejecutamos la conversión...
    Ok((StatusCode::OK, "Archivo recibido").into_response())
}
```

Salida esperada al compilar el código base:
```text
   Compiling ffmpeg-api-post v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 2.34s
```

### Paso 3: Conversión asíncrona con FFmpeg

Invocamos a FFmpeg como un subproceso del sistema sin detener el bucle de eventos (event loop) de Tokio. Usamos `tokio::process::Command`.

```rust
use tokio::process::Command;

async fn run_ffmpeg(input: &str, output: &str) -> Result<(), String> {
    let status = Command::new("ffmpeg")
        .args([
            "-y",
            "-i", input,
            "-acodec", "libmp3lame",
            "-ab", "128k",
            "-ar", "16000",
            output
        ])
        .status()
        .await
        .map_err(|e| e.to_string())?;

    if status.success() {
        Ok(())
    } else {
        Err("FFmpeg proceso fallido".to_string())
    }
}
```

Salida esperada al ejecutar una prueba interna del proceso:
```text
[info] FFmpeg comando ejecutado con éxito en 45ms.
```

### Paso 4: Transmitir el resultado en stream

Abrimos el archivo convertido y lo transmitimos en trozos pequeños directamente al cliente HTTP. Esto mantiene la memoria RAM limpia.

```rust
use axum::body::Body;
use tokio_util::io::ReaderStream;

async fn stream_file(path: &str) -> Result<Response, StatusCode> {
    let file = File::open(path).await.map_err(|_| StatusCode::NOT_FOUND)?;
    let stream = ReaderStream::new(file);
    let body = Body::from_stream(stream);

    Ok(Response::builder()
        .header("Content-Type", "audio/mp3")
        .body(body)
        .unwrap())
}
```

Salida HTTP esperada al recibir el stream:
```text
HTTP/1.1 200 OK
Content-Type: audio/mp3
Transfer-Encoding: chunked
```

## Errores comunes al integrar FFmpeg

El error más grave es ejecutar `std::process::Command` en lugar de la versión de `tokio`. Esto bloquea el hilo de ejecución principal, impidiendo que Axum atienda otras peticiones en paralelo.

Otro fallo clásico es cargar todo el archivo MP3 en memoria antes de responder. Si diez usuarios suben audios pesados al mismo tiempo, el servidor se queda sin RAM de inmediato. Usa siempre `ReaderStream` para enviar datos de forma progresiva.

Finalmente, recuerda programar limpiezas periódicas de los archivos temporales. La carpeta `/tmp` puede llenarse rápidamente en entornos con mucho tráfico.

## Resumen del aprendizaje

*   La API de Whisper rinde mejor con archivos `.mp3` optimizados a 128kbps y 16kHz.
*   Rust y Axum manejan las peticiones concurrentes con latencia mínima.
*   `tokio::process::Command` evita bloquear los hilos principales de la aplicación.
*   `ReaderStream` mantiene el uso de memoria RAM constante y plano.

Como siguiente paso, puedes empaquetar tu binario en un contenedor Docker ligero que incluya FFmpeg preinstalado y subirlo a producción.
