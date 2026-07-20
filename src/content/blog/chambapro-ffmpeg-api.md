---
title: 'Procesamiento de Audio en Tiempo Real con Rust y FFmpeg'
description: 'Cómo construí un microservicio ultra rápido y escalable en Rust para convertir audios de WhatsApp en milisegundos y alimentar pipelines de OpenAI Whisper.'
pubDate: 'Jul 19 2026'
heroImage: 'https://opengraph.githubassets.com/8ec9919eeb7baeab4090eaed242cd15033af0fb105f48a60e996f75f63930a63/rrortega/chambapro-ffmpeg-api'
---

Integrar bots conversacionales inteligentes con canales de mensajería populares (como WhatsApp o Facebook Messenger) presenta desafíos técnicos interesantes. Uno de los problemas más comunes ocurre cuando los usuarios envían notas de voz. 

Plataformas como ManyChat, yCloud o Kaypso entregan estos audios en formatos optimizados para mensajería como `.oga` o `.ogg`. Sin embargo, los motores de Speech-to-Text (STT) modernos, como **OpenAI Whisper**, son sustancialmente más rápidos y precisos cuando procesan archivos `.mp3` estándar.

Para resolver este cuello de botella, diseñé y construí **[chambapro-ffmpeg-api](https://github.com/rrortega/chambapro-ffmpeg-api)**: un microservicio ultra ligero y concurrente escrito en Rust que actúa como proxy de conversión en tiempo real.

---

## 📊 Arquitectura y Flujo de Trabajo

El servicio está diseñado para funcionar en dos modos de ejecución, dependiendo de la infraestructura disponible:

### Modo 1: Procesamiento Directo (Sin Redis)
Ideal para despliegues sencillos. El servidor recibe el archivo o la URL, inicia un hilo en background para procesar la conversión con FFmpeg, transmite el archivo binario convertido directamente al Webhook configurado e inmediatamente limpia el almacenamiento local.

### Modo 2: Cola Distribuida (Con Redis)
Para escenarios de alta concurrencia y tráfico masivo. El flujo funciona de la siguiente manera:

1. **Recepción**: El servidor web de Axum recibe una petición `POST /convert-async` con el archivo o URL y un `callback_url`.
2. **Encolamiento**: La API inserta el trabajo en una cola de Redis (`LPUSH`) y responde de inmediato al cliente con un `202 Accepted` y el UUID del trabajo.
3. **Procesamiento**: Los background workers consumen los trabajos de la cola, ejecutan FFmpeg de manera aislada y guardan el resultado.
4. **Notificación**: Se envía una petición HTTP `POST` al webhook con el resultado.
5. **Autolimpieza**: Se programa una tarea retrasada en Redis para eliminar los archivos generados después de 24 horas.

---

## 🛠️ Stack Tecnológico

Elegí cada componente de este stack buscando el máximo rendimiento y estabilidad:

- **Rust & Axum**: Rust garantiza seguridad en memoria y concurrencia sin recolector de basura (garbage collector). Axum nos provee un enrutador HTTP de alta velocidad basado en Tokio.
- **FFmpeg**: El motor estándar de la industria para manipulación multimedia, invocado eficientemente mediante subprocesos controlados.
- **Redis**: Como broker de mensajería y almacenamiento de estados, usando listas y conjuntos ordenados para las colas y las tareas retrasadas.
- **Tokio & ReaderStream**: Para transmitir los archivos binarios trozo a trozo (chunk-by-chunk) hacia los clientes o webhooks, manteniendo el uso de memoria RAM completamente plano.

---

## 🚀 Ejemplos de Uso

### Conversión Síncrona (Upload directo)
Podés enviar un archivo de audio directamente y recibir el flujo binario procesado en la respuesta:

```bash
curl -X POST http://localhost/convert \
  -F "file=@input.oga" \
  -F "output_format=mp3" \
  --output output.mp3
```

### Conversión Asíncrona (Basado en colas)
Para procesar archivos pesados o integraciones webhooks:

```bash
curl -X POST http://localhost/convert-async \
  -F "url=https://example.com/audio.oga" \
  -F "output_format=mp3" \
  -F "callback_url=https://your-webhook.com/callback"
```

El servicio responderá inmediatamente:

```json
{
  "uuid": "7a94dfbd-5b0c-4464-9b2f-3b2d6a5c2f9d",
  "enqueue": true
}
```

---

## 📈 Dashboard en Tiempo Real

El servicio no es solo una API invisible; expone un panel web interactivo (`GET /dashboard`) donde se pueden monitorear las métricas de la cola (pendientes, exitosos, fallados), ver actualizaciones en tiempo real de los trabajos y visualizar los logs del proceso en vivo.

Si te interesa revisar el código, extender el servicio o desplegarlo en tu propia infraestructura con Docker, podés acceder al repositorio oficial en GitHub: 

👉 **[rrortega/chambapro-ffmpeg-api](https://github.com/rrortega/chambapro-ffmpeg-api)**
