---
title: 'OmniRoute: Gateway de IA auto-hospedado en tu VPS'
description: 'Instala OmniRoute en un VPS de Hostinger para unificar proveedores de IA, balancear consumo y evitar límites de cuota gratis.'
pubDate: 'Jul 20 2026'
heroImage: '../../assets/tier-cascade.svg'
---

¿Te ha pasado que estás programando y de repente te quedas sin saldo en OpenAI o golpeas el límite de cuota en Anthropic? Si utilizas herramientas de asistencia de IA en tu día a día, depender de un único proveedor es una receta segura para la frustración y la improductividad. 

## El puente agnóstico que tu flujo de trabajo necesita

Estoy completamente enamorado de este proyecto. **OmniRoute** es un gateway de IA gratuito y self-hosted que te permite unificar hasta 268 proveedores de inferencia en un solo endpoint local o remoto. 

La magia es simple pero brutal: en lugar de configurar credenciales individuales en cada uno de tus IDEs o asistentes de codificación (como Cursor, Claude Code o Antigravity), apuntas todo hacia tu propia instancia de OmniRoute. El gateway se encarga de balancear tus peticiones usando 18 estrategias de enrutamiento avanzadas: round-robin, minimización de costos, peso dinámico, o cascadas automáticas con fallbacks inmediatos en milisegundos. Si un proveedor falla, el siguiente toma el control de forma transparente.

Además, incluye compresión de tokens (RTK + Caveman stacked) que te ayuda a ahorrar desde un 15% hasta un 95% del consumo en sesiones con mucho contexto, observabilidad en tiempo real, alertas de presupuesto y caché semántica.

---

## Paso 1: Preparar tu servidor en Hostinger

Para garantizar disponibilidad 24/7 y una latencia ultra baja, lo ideal es hospedar tu gateway en un servidor privado virtual. Te recomiendo levantar una máquina virtual limpia con Ubuntu o Debian usando un [VPS de Hostinger](https://www.hostinger.com?REFERRALCODE=LVIWINFOLVXC), que ofrece un rendimiento excelente a un precio sumamente competitivo.

Una vez que tengas tu servidor activo en tu panel de control, conéctate a él mediante SSH desde tu terminal favorita:

```bash
ssh root@IP_DE_TU_VPS
```

Una vez dentro, actualiza la lista de paquetes del sistema para asegurarte de tener todo al día:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Paso 2: Instalar Node.js y OmniRoute

OmniRoute está construido sobre Node.js, por lo que requerimos instalarlo de manera global. La forma más limpia de hacerlo es usando NVM (Node Version Manager) para evitar problemas de permisos con el gestor de paquetes nativo:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20
```

Con Node.js activo en la versión 20, ya puedes proceder a instalar la interfaz de comandos (CLI) de OmniRoute de forma global en tu sistema:

```bash
npm install -g omniroute
```

Una vez instalada, lanza el servicio escribiendo simplemente:

```bash
omniroute
```

Verás que el servicio levanta el panel de administración y el proxy API en el puerto `20128`:

```text
▸ dashboard ✓ http://localhost:20128/dashboard
▸ api ......✓ serving on :20128
```

---

## Paso 3: Conexión remota segura desde tu laptop

La genialidad de OmniRoute radica en su **Remote Mode**. Puedes instalar el CLI en tu computadora local para controlar la instancia que acabas de configurar en tu servidor de [Hostinger](https://www.hostinger.com?REFERRALCODE=LVIWINFOLVXC).

Primero, instala el cliente localmente en tu laptop:

```bash
npm install -g omniroute
```

Ahora, conéctate a tu VPS utilizando tu dirección IP pública. El sistema te pedirá ingresar la contraseña de administración que configuraste en tu primer inicio:

```bash
omniroute connect IP_DE_TU_VPS
```

A partir de este momento, cualquier comando que ejecutes en la terminal de tu laptop (por ejemplo, `omniroute models list` o `omniroute configure`) se ejecutará directamente contra tu VPS de manera remota.

---

## Paso 4: Vincular Antigravity o Claude Code mediante OAuth

Si intentas conectar herramientas que usan autenticación de Google (como Antigravity) en un servidor remoto de forma directa, la pantalla de consentimiento se quedará congelada. Esto sucede porque Google requiere que la redirección de bucle local (`127.0.0.1`) responda en el mismo navegador que aprueba el inicio de sesión.

Para solucionar esto, OmniRoute incluye un helper de inicio de sesión local muy conveniente:

1. Ejecuta el helper en tu **laptop local**:
   ```bash
   npx omniroute login antigravity
   ```
2. Completa el flujo de inicio de sesión en tu navegador habitual. La terminal local generará un bloque de credenciales seguro que se ve así:
   `omniroute-cred-v1.eyJ2IjoxLCJ...`
3. Entra al panel de control remoto de tu VPS (`http://IP_DE_TU_VPS:20128/dashboard`), ve a **Providers** -> **Antigravity** -> **Connect** y pega ese bloque directamente en el campo del Paso 2.

¡Listo! El gateway guardará el token encriptado en el servidor y tus asistentes de código podrán usarlo de inmediato.

---

## Problemas comunes y cómo solucionarlos

*   **Error de conexión (Timeout)**: Asegúrate de que el puerto `20128` esté abierto en el firewall de tu VPS de [Hostinger](https://www.hostinger.com?REFERRALCODE=LVIWINFOLVXC). Puedes habilitarlo temporalmente con `ufw allow 20128/tcp`.
*   **Huelga de tokens de Google**: Si el token expira, no intentes re-autenticar desde la consola del VPS. Vuelve a correr el login helper `npx omniroute login antigravity` en tu máquina local y actualiza la clave en el panel.

---

## Conclusión y siguientes pasos

Tener tu propio enrutador de modelos de lenguaje te da libertad absoluta. Puedes crear un modelo virtual llamado `auto` que balancee dinámicamente tus peticiones según costo o disponibilidad.

*   Unifica tus suscripciones en un solo endpoint.
*   Ahorra presupuesto aprovechando las capas de uso gratuito de proveedores como Kiro o LongCat.
*   Protege tus herramientas contra caídas de servicio de un solo backend.

<div class="cta-note">
  <strong>💡 Nota del autor:</strong>
  Machete, si crees que esto te es útil, date una vuelta por GitHub y déjale una estrella al repositorio oficial. Ya te ahorré la "chamba" de buscarlo. Si quieres levantar tu propia instancia en minutos con un solo comando, puedes usar el contenedor oficial de Docker:
  <br/><br/>
  <code>docker pull diegosouzapw/omniroute:latest</code>
  <br/><br/>
  👉 Repositorio oficial: <a href="https://github.com/diegosouzapw/OmniRoute" target="_blank" rel="noopener noreferrer">github.com/diegosouzapw/OmniRoute</a>
</div>
