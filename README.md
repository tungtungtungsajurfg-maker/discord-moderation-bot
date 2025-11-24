# discord-moderation-bot
Es mi bot de Discord 

🚀 Características principales

🔨 Moderación automática

Filtros de toxicidad

Detección de contenido ofensivo (texto)

Anti-spam

Anti-raid


👮‍♂️ Comandos de staff

ban, kick, mute, unmute, warn


📜 Sistema de logs

Ingreso de nuevos usuarios

Acciones de moderadores

Alertas en tiempo real


⚙️ Totalmente configurable Desde variables de entorno en Railway.



---

🔧 Requisitos

Tu bot usa estas librerías:

pip install discord.py
pip install aiohttp
pip install python-dotenv
pip install sqlite3-binary
pip install asyncio


---

🛠️ Configuración en Railway

En Variables agrega:

Variable	Descripción

TOKEN	Token del bot (NO lo subas a GitHub)
CANAL_ALERTAS	ID del canal de alertas (1442022471405670521)
CANAL_LOGS	ID del canal de logs de nuevos usuarios (1442027971853418527)
ROL_SILENCIO	ID del rol para mutear (1441956468810186885)



---

🚀 Iniciar bot en Railway

El Procfile debe contener:

worker: python3 bot.py


---

📝 Notas importantes

❗ Nunca subas tu token a GitHub

✔️ Todo el código sensible debe ir en variables de entorno

🔒 Tu bot estará protegido contra ataques básicos de seguridad
