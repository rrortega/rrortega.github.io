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

## El flujo de datos y la arquitectura

El sistema cuenta con un flujo diseñado para garantizar la velocidad de procesamiento. Se divide en dos modos de ejecución según las necesidades de tu infraestructura.

El siguiente diagrama muestra cómo viajan los datos asíncronamente desde el cliente hasta el webhook final:

![Diagrama de Arquitectura](https://mermaid.ink/img/pako:eJytVNtuGjEQ_ZXRPlQgQWlRn1C0Eg2NSkVVygblJVI0aw9g4fVsfCFBUT4m39Ifq8zuBpqE8FI_re3jOWeOj_chESwpGSSObgMZQSOFS4vFtQEAwODZhCInW8-FZwvnWpHx1UqJ1iuhSjQehtMxoIPhfSjginLIyG6ak4e434ECReSMpHL1tDWZzrPv7dfoK7ZrshH-FcV6aTkYWS-61-jMs8XlrvqF0tTM3yhL-Yp5HYHN5yXaJflrU4GrLrtpOpyOBzD9lV1CT7DZkPVddFsjoBUZevPZpAMCtc5RrG-C1XUPqD3Mvo3G2c18NgFy_s8TCDYLtQwWJUPrJ0uGfo2OYzgdd9N0Z8cApsGt4HzH5xQb-ME5tObz8ejFgW6aVkoH0P_Uh6EQVHqS8AAhKNkBMrdVQW8DweP-sGYuG3MnzOV-J45q40AOl1HCEVDt8gBmwYDYi25dXBQlLdvwARxuCDyDq6C9sygv_XhG9z59nzoTK5JBE5xrQhPKyor-lxVI0rgl2T4hPTrZ3PE_PZCRL_2oYSPSakN2e6RyDatj4Tz64HqS74xmlDED0GILuTJot1DiNi633-MdVZ0892pPGT0iTZ7g0MXKE42e7FtcpB0dBNLw0Ux-fpXJ3RPISrwzkO8foV9ZQvmf4lgRnQrSS_zzRWTeEhaN5Yv49D0313mUZVwUJBV6AlFnixc1bRQdy9TmRReTTlKQLVDJZPCQ-BUV8ccpaYFB--Tx8S_saq3p)

En el Modo 1, la API maneja todo en memoria y subprocesos internos inmediatos. En el Modo 2, Redis gestiona la distribución del trabajo, permitiendo absorber picos masivos de tráfico mediante workers dedicados.

## Resiliencia y escalabilidad en producción

En el mundo real, los archivos de audio pueden estar corruptos, la red puede fallar a mitad de la descarga, o el servidor del webhook destino puede caerse temporalmente. Por esto, la resiliencia y la tolerancia a fallos son pilares fundamentales del diseño.

### Manejo de reintentos
Si una conversión o entrega falla dentro de la cola, el sistema no descarta el trabajo inmediatamente. El worker reintenta el proceso de manera automática hasta un máximo configurable (`MAX_RETRIES`, por defecto 3) antes de reportar un estado fallido definitivo.

### Predicibilidad de errores
Cada etapa del proceso actualiza su estado en Redis (`Enqueued`, `Processing`, `Success`, `Failed`). Si la conversión falla, los detalles específicos del error quedan registrados y pueden consultarse directamente mediante el endpoint `/status/:uuid`, facilitando el monitoreo continuo.

### Autolimpieza preventiva
Para prevenir que el disco del servidor se llene con audios antiguos y provoque una caída general, implementé un sistema de limpieza automática basado en conjuntos ordenados de Redis. Los archivos convertidos se borran permanentemente 24 horas después de su creación.

## Paso 1: Configurar el entorno de Rust

Iniciamos un nuevo proyecto binario usando Cargo en la terminal.

```bash
cargo new ffmpeg-api-post
cd ffmpeg-api-post
```

Salida esperada de la terminal:
```text
     Created binary (application) `ffmpeg-api-post` package
```

Configuramos las dependencias necesarias en el archivo `Cargo.toml` para soportar Axum y tareas de entrada/salida asíncronas.

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1.35", features = ["full"] }
tokio-util = { version = "0.7", features = ["io"] }
uuid = { version = "1.6", features = ["v4"] }
```

## Paso 2: Servidor base y recepción de archivos

Configuramos el enrutador de Axum para aceptar flujos de archivos utilizando peticiones multipart en el handler `/convert`.

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
    
    Ok((StatusCode::OK, "Archivo recibido").into_response())
}
```

Salida esperada al compilar el código base:
```text
   Compiling ffmpeg-api-post v0.1.0
    Finished dev [unoptimized + debuginfo] target(s) in 2.34s
```

## Paso 3: Conversión asíncrona con FFmpeg

Invocamos a FFmpeg como un subproceso del sistema sin detener el bucle de eventos de Tokio usando `tokio::process::Command`.

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

Salida esperada al ejecutar la conversión:
```text
[info] FFmpeg comando ejecutado con éxito en 45ms.
```

## Paso 4: Transmitir el resultado en stream

Abrimos el archivo convertido y lo transmitimos en trozos pequeños directamente en la respuesta HTTP para evitar picos de memoria RAM.

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

## Monitoreo con Dashboard Visual

La observabilidad es clave en sistemas productivos. El servicio expone un panel web interactivo e interactivo en `/dashboard` que permite monitorear el estado de la cola en vivo.

![Chambapro FFmpeg Dashboard](https://raw.githubusercontent.com/rrortega/chambapro-ffmpeg-api/main/dashboard.png)

Este panel muestra las métricas de la cola (pendientes, procesados, fallados), actualizaciones de trabajos en tiempo real mediante Server-Sent Events (SSE) y una transmisión en directo de la salida estándar (stdout) del proceso.

## Errores comunes al integrar FFmpeg

El error más grave es ejecutar `std::process::Command` en lugar de la versión de `tokio`. Esto bloquea el hilo de ejecución principal, impidiendo que Axum atienda otras peticiones en paralelo.

Otro fallo clásico es cargar todo el archivo MP3 en memoria antes de responder. Si diez usuarios suben audios pesados al mismo tiempo, el servidor se queda sin RAM de inmediato. Usa siempre `ReaderStream` para enviar datos de forma progresiva.

## Resumen del aprendizaje

*   El desacoplamiento con colas de Redis previene sobrecargas del servidor HTTP bajo tráfico masivo.
*   El manejo de reintentos y estados claros del trabajo incrementa la resiliencia en producción.
*   `tokio::process::Command` evita bloquear los hilos principales de la aplicación.
*   La limpieza automática previene interrupciones causadas por falta de espacio en disco.

<div class="cta-note">

💡 **Machete**, si crees que esto te es útil pásate por GitHub y déjame una estrella. Ya te hice la "pinchamba" y tienes también la imagen lista de Docker Hub para que le hagas pull y la instales con un solo comando:

```bash
docker pull rrortega/chambapro-ffmpeg-api:latest
```

👉 **[github.com/rrortega/chambapro-ffmpeg-api](https://github.com/rrortega/chambapro-ffmpeg-api)**

</div>
