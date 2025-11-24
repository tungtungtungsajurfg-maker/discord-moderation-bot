
import os
import time
import hashlib
import aiohttp
import discord
from discord.ext import commands, tasks
from collections import defaultdict
from datetime import datetime
import sqlite3
import asyncio

# ===================== CONFIGURACIÓN (MODIFICA ESTO) =====================

TOKEN = ""  # <- reemplaza con tu token real

OWNER_ID = 1315133497924391004  # tu ID numérico (sin comillas)

# Canales (usa IDs numéricas)
ALERT_CHANNEL_ID = 1442022471405670521      # canal de alertas (donde se publica después de owner)
LOG_CHANNEL_ID   = 1442027971853418527      # canal de logs / nuevos usuarios
STAFF_ROLE_ID    = 1442310490146738339      # rol staff (puede usar comandos moderación)
MUTE_ROLE_ID     = 1441956468810186885      # ID del rol 'Muted' (si existe). Si no existe, lo creará.
# Rol que reciben nuevos usuarios (puedes usar ID o dejar None y usar nombre)
ROL_NUEVO_ID = 1442243131021070346  # si lo tienes como ID (int) o None
ROL_NUEVO_NAME = "🔥Chaval💥"

# NSFW - imágenes solamente
NSFW_ENABLED = True
NSFW_THRESHOLD = 0.6
NSFW_HIGH_THRESHOLD = 0.95

# Rutas y límites
TEMP_DIR = "tmp_moderator"
EVIDENCE_DIR = "evidencia"
DB_PATH = "data/moderator.sqlite3"
MAX_ATTACHMENT_MB = 50

# Anti-spam / flood
FLOOD_WINDOW_SECONDS = 5
FLOOD_MESSAGES_THRESHOLD = 5
FLOOD_PURGE_LIMIT = 50

# ===================== DEPENDENCIAS NSFW =====================

try:
    from nsfw_detector import predict
except Exception:
    predict = None

# ===================== UTILIDADES =====================

def ensure_dirs():
    os.makedirs(TEMP_DIR, exist_ok=True)
    os.makedirs(EVIDENCE_DIR, exist_ok=True)
    os.makedirs(os.path.dirname(DB_PATH), exist_ok=True)

async def save_attachment_to_temp(att, temp_dir=TEMP_DIR):
    os.makedirs(temp_dir, exist_ok=True)
    safe_name = f"{int(time.time())}_{att.id}_{att.filename}"
    local = os.path.join(temp_dir, safe_name)
    async with aiohttp.ClientSession() as session:
        async with session.get(att.url) as resp:
            cl = resp.headers.get("content-length")
            if cl and int(cl) > MAX_ATTACHMENT_MB * 1024 * 1024:
                raise ValueError("Archivo excede tamaño máximo")
            data = await resp.read()
            if len(data) > MAX_ATTACHMENT_MB * 1024 * 1024:
                raise ValueError("Archivo excede tamaño máximo")
            with open(local, "wb") as f:
                f.write(data)
    return local

def sha1_bytes(data: bytes):
    return hashlib.sha1(data).hexdigest()

def save_evidence(author_name, filename, temp_path):
    fecha = datetime.now().strftime("%Y-%m-%d")
    user_folder = os.path.join(EVIDENCE_DIR, fecha, author_name)
    os.makedirs(user_folder, exist_ok=True)
    dest_name = f"{int(time.time())}_{os.path.basename(filename)}"
    dest_path = os.path.join(user_folder, dest_name)
    try:
        os.replace(temp_path, dest_path)
    except Exception:
        import shutil
        shutil.copy2(temp_path, dest_path)
    return dest_path

# Base de datos simple para warns
def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""
    CREATE TABLE IF NOT EXISTS warns (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        guild_id TEXT,
        user_id TEXT,
        moderator_id TEXT,
        reason TEXT,
        ts INTEGER
    )""")
    conn.commit()
    conn.close()

def add_warn(guild_id, user_id, moderator_id, reason):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("INSERT INTO warns (guild_id,user_id,moderator_id,reason,ts) VALUES (?,?,?,?,?)",
              (str(guild_id), str(user_id), str(moderator_id), reason, int(time.time())))
    conn.commit()
    conn.close()

def get_warns_for_user(guild_id, user_id):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT id, moderator_id, reason, ts FROM warns WHERE guild_id=? AND user_id=?",
              (str(guild_id), str(user_id)))
    rows = c.fetchall()
    conn.close()
    return rows

# ===================== MÓDULOS =====================

class NSFWModule:
    def __init__(self, core):
        self.core = core
        self.model = None
        if NSFW_ENABLED and predict:
            path = os.getenv("NSFW_MODEL_PATH")
            if path and os.path.exists(path):
                try:
                    self.model = predict.load_model(path)
                    print("[NSFW] Modelo cargado:", path)
                except Exception as e:
                    print("[NSFW] Error cargando modelo:", e)
            else:
                print("[NSFW] nsfw_detector disponible pero NSFW_MODEL_PATH no definido o no existe.")
        else:
            if NSFW_ENABLED:
                print("[NSFW] nsfw_detector no instalado o no disponible. NSFW deshabilitado.")
            else:
                print("[NSFW] NSFW deshabilitado por configuración.")

    async def handle_message(self, message):
        # Procesa adjuntos de imagen tanto en canales como en DMs
        if not message.attachments:
            return False
        # Si es DM y detección en DMs: lo permitimos (usuario lo solicitó)
        for att in message.attachments:
            try:
                ext = os.path.splitext(att.filename)[1].lower()
                if ext not in (".jpg", ".jpeg", ".png", ".webp"):
                    continue
                local = await save_attachment_to_temp(att)
                with open(local, "rb") as f:
                    h = sha1_bytes(f.read())
                # cache simple
                if h in self.core.result_cache:
                    res = self.core.result_cache[h]
                    nsfw_result = res.get("nsfw")
                    evidence = res.get("evidence")
                else:
                    nsfw_result = None
                    evidence = local
                    if self.model:
                        preds = predict.classify(self.model, local)
                        inner = preds.get(local) if isinstance(preds, dict) else preds
                        if inner:
                            top = max(inner, key=inner.get)
                            prob = float(inner[top])
                            nsfw_result = {"label": top, "prob": prob}
                    self.core.result_cache[h] = {"nsfw": nsfw_result, "evidence": evidence, "ts": time.time()}

                if nsfw_result:
                    prob = nsfw_result.get("prob", 0.0)
                    label = nsfw_result.get("label", "")
                    severity = "alta" if prob >= NSFW_HIGH_THRESHOLD else ("media" if prob >= NSFW_THRESHOLD else "baja")
                    ev_path = save_evidence(str(message.author.id), att.filename, evidence)
                    # Acción según severidad: media -> mute, alta -> ban
                    if severity == "alta":
                        # ban inmediato
                        try:
                            if message.guild:
                                await message.guild.ban(message.author, reason="Detección NSFW alta", delete_message_days=0)
                        except Exception as e:
                            print("[NSFW] Error baneando:", e)
                        await self.core.dispatch_alert(message.guild or None, message, message.author,
                                                       f"Detección NSFW ALTA: {label}", severity, label, prob, ev_path)
                        return True
                    elif severity == "media":
                        # mute automático (intentar con rol por ID)
                        try:
                            if message.guild:
                                mute_role = message.guild.get_role(int(MUTE_ROLE_ID)) if MUTE_ROLE_ID else None
                                if not mute_role:
                                    # crear role Muted si no existe
                                    mute_role = await message.guild.create_role(name="Muted", reason="Rol auto para mute")
                                    # negar enviar mensajes en todos los canales
                                    for ch in message.guild.channels:
                                        try:
                                            await ch.set_permissions(mute_role, send_messages=False, add_reactions=False)
                                        except Exception:
                                            pass
                                await message.author.add_roles(mute_role, reason="Mute automático por NSFW (media)")
                        except Exception as e:
                            print("[NSFW] Error aplicando mute:", e)
                        await self.core.dispatch_alert(message.guild or None, message, message.author,
                                                       f"Detección NSFW MEDIA: {label} (mute aplicado)", severity, label, prob, ev_path)
                        return True
                    else:
                        # baja - aviso y log, no sanción automática
                        await self.core.dispatch_alert(message.guild or None, message, message.author,
                                                       f"Detección NSFW: {label}", severity, label, prob, ev_path)
                        return True
            except Exception as e:
                print("[NSFW] error:", e)
        return False

class RaidModule:
    def __init__(self, core):
        self.core = core
        self.last_messages = defaultdict(list)

    async def handle_message(self, message):
        if not message.guild:
            return False
        key = (message.guild.id, message.channel.id, message.author.id)
        now = time.time()
        arr = self.last_messages[key]
        arr.append(now)
        while len(arr) > 100:
            arr.pop(0)
        recent = [t for t in arr if now - t <= FLOOD_WINDOW_SECONDS]
        if len(recent) >= FLOOD_MESSAGES_THRESHOLD:
            try:
                deleted = await message.channel.purge(limit=FLOOD_PURGE_LIMIT, check=lambda m: m.author.id == message.author.id)
                await self.core.dispatch_alert(message.guild, message, message.author,
                                              f"Flood/spam detectado: {len(deleted)} mensajes eliminados",
                                              "media", "spam", 1.0, None)
                return True
            except Exception as e:
                print("[RaidModule] Error purgando:", e)
        return False

# ===================== BOT CORE =====================

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
intents.guilds = True
intents.messages = True

bot = commands.Bot(command_prefix="!", intents=intents)

# Attach modules and cache
bot.result_cache = {}
bot.nsfw_module = NSFWModule(bot)
bot.raid_module = RaidModule(bot)

ensure_dirs()
init_db()

# ===== Utilidad: obtener canal de alertas por ID preferido =====
def get_channel_by_id_or_name(guild, id_val, name_val):
    try:
        if id_val:
            ch = guild.get_channel(int(id_val))
            if ch:
                return ch
    except Exception:
        pass
    for c in guild.text_channels:
        if c.name == str(name_val):
            return c
    return None

async def send_owner_then_channel(guild, message, author, title, severity, label, prob, evidence_path):
    # notificar owner
    try:
        owner_user = await bot.fetch_user(int(OWNER_ID))
        if owner_user:
            embed = discord.Embed(
                title=f"🚨 {title}",
                description=f"Usuario: {author}\nSeveridad: **{severity}**\nEtiqueta: `{label}`\nProbabilidad: `{prob}`",
                color=discord.Color.red(),
                timestamp=datetime.utcnow()
            )
            if evidence_path and os.path.exists(evidence_path):
                try:
                    await owner_user.send(embed=embed, file=discord.File(evidence_path))
                except Exception:
                    await owner_user.send(embed=embed)
            else:
                await owner_user.send(embed=embed)
    except Exception as e:
        print("[ALERT] no se pudo notificar owner:", e)

    # notificar canal de alertas
    try:
        if message and message.guild:
            ch = get_channel_by_id_or_name(message.guild, ALERT_CHANNEL_ID, None)
            if ch:
                embed2 = discord.Embed(
                    title=f"⚠️ {title}",
                    description=f"Usuario: {author.mention}\nSeveridad: **{severity}**\nEtiqueta: `{label}`\nProbabilidad: `{prob}`",
                    color=discord.Color.red(),
                    timestamp=datetime.utcnow()
                )
                if evidence_path and os.path.exists(evidence_path):
                    try:
                        await ch.send(embed=embed2, file=discord.File(evidence_path))
                    except Exception:
                        await ch.send(embed=embed2)
                else:
                    await ch.send(embed=embed2)
    except Exception as e:
        print("[ALERT] no se pudo enviar a canal:", e)

async def dispatch_alert(guild, message, author, title, severity, label, prob, evidence):
    await send_owner_then_channel(guild, message, author, title, severity, label, prob, evidence)
    # intentar borrar mensaje para no incomodar
    try:
        if message and message.guild:
            await message.delete()
    except Exception:
        pass

bot.dispatch_alert = dispatch_alert

# ===================== EVENT HANDLERS =====================

@bot.event
async def on_ready():
    print("Bot listo:", bot.user)
    cleanup_cache.start()

@bot.event
async def on_message(message):
    if message.author.bot:
        return
    try:
        handled = await bot.raid_module.handle_message(message)
        if handled:
            return
    except Exception as e:
        print("[on_message] raid error:", e)
    try:
        handled = await bot.nsfw_module.handle_message(message)
        if handled:
            return
    except Exception as e:
        print("[on_message] nsfw error:", e)
    await bot.process_commands(message)

@bot.event
async def on_member_join(member):
    try:
        role = None
        if ROL_NUEVO_ID:
            role = member.guild.get_role(int(ROL_NUEVO_ID))
        if not role:
            role = discord.utils.get(member.guild.roles, name=ROL_NUEVO_NAME)
        if role:
            await member.add_roles(role, reason="Asignación automática de bienvenida")
    except Exception as e:
        print("[join] error asignando rol:", e)
    try:
        canal = member.guild.system_channel
        if canal:
            await canal.send(f"Bienvenido/a {member.mention}! Se te ha asignado automáticamente el rol '{role.name if role else ROL_NUEVO_NAME}'.")
    except Exception:
        pass

@bot.event
async def on_member_remove(member):
    try:
        canal = member.guild.system_channel
        if canal:
            await canal.send(f"El usuario {member.name} ha salido del servidor.")
    except Exception:
        pass

# ===================== COMANDOS DE MODERACIÓN =====================

def is_staff_or_owner():
    async def predicate(ctx):
        if ctx.author.id == int(OWNER_ID):
            return True
        # si tiene rol staff por ID
        try:
            if STAFF_ROLE_ID and any(r.id == int(STAFF_ROLE_ID) for r in ctx.author.roles):
                return True
        except Exception:
            pass
        perms = ctx.author.guild_permissions
        return perms.manage_messages or perms.kick_members or perms.ban_members or perms.administrator
    return commands.check(predicate)

@bot.command(name="ban")
@is_staff_or_owner()
async def cmd_ban(ctx, member: discord.Member, *, reason: str = "No especificada"):
    try:
        await member.ban(reason=reason)
        await ctx.send(f"✅ {member} ha sido baneado. Razón: {reason}")
        await dispatch_alert(ctx.guild, ctx.message, member, f"Usuario baneado: {member}", "alta", "ban", 1.0, None)
    except Exception as e:
        await ctx.send(f"❌ Error al banear: {e}")

@bot.command(name="kick")
@is_staff_or_owner()
async def cmd_kick(ctx, member: discord.Member, *, reason: str = "No especificada"):
    try:
        await member.kick(reason=reason)
        await ctx.send(f"✅ {member} ha sido expulsado. Razón: {reason}")
        await dispatch_alert(ctx.guild, ctx.message, member, f"Usuario expulsado: {member}", "media", "kick", 1.0, None)
    except Exception as e:
        await ctx.send(f"❌ Error al expulsar: {e}")

@bot.command(name="clear")
@is_staff_or_owner()
async def cmd_clear(ctx, amount: int = 10):
    try:
        deleted = await ctx.channel.purge(limit=amount+1)
        await ctx.send(f"🧹 Se eliminaron {len(deleted)-1} mensajes.", delete_after=6)
    except Exception as e:
        await ctx.send(f"❌ Error al limpiar: {e}")

@bot.command(name="warn")
@is_staff_or_owner()
async def cmd_warn(ctx, member: discord.Member, *, reason: str = "No especificada"):
    add_warn(ctx.guild.id, member.id, ctx.author.id, reason)
    await ctx.send(f"⚠️ {member.mention} ha recibido una advertencia. Razón: {reason}")
    await dispatch_alert(ctx.guild, ctx.message, member, f"Advertencia a {member}", "media", "warn", 1.0, None)

@bot.command(name="warns")
@is_staff_or_owner()
async def cmd_warns(ctx, member: discord.Member):
    rows = get_warns_for_user(ctx.guild.id, member.id)
    if not rows:
        await ctx.send(f"No hay advertencias para {member}.")
        return
    msg = [f"ID:{r[0]} Moderador:{r[1]} Razón:{r[2]} Fecha:{datetime.fromtimestamp(r[3]).strftime('%Y-%m-%d %H:%M:%S')}" for r in rows]
    await ctx.send("Advertencias:\n" + "\n".join(msg))

@bot.command(name="mute")
@is_staff_or_owner()
async def cmd_mute(ctx, member: discord.Member, minutos: int = 0, *, reason: str = "No especificada"):
    guild = ctx.guild
    muted_role = guild.get_role(int(MUTE_ROLE_ID)) if MUTE_ROLE_ID else None
    if not muted_role:
        try:
            muted_role = await guild.create_role(name="Muted", reason="Rol para mute")
            for ch in guild.channels:
                try:
                    await ch.set_permissions(muted_role, send_messages=False, add_reactions=False)
                except Exception:
                    pass
        except Exception as e:
            await ctx.send(f"❌ Error creando rol Muted: {e}")
            return
    try:
        await member.add_roles(muted_role, reason=reason)
        await ctx.send(f"🔇 {member.mention} ha sido silenciado. {('Duración: %d min.' % minutos) if minutos>0 else ''}")
        await dispatch_alert(guild, ctx.message, member, f"Usuario silenciado: {member}", "media", "mute", 1.0, None)
        if minutos > 0:
            async def unmute_later():
                await asyncio.sleep(minutos * 60)
                try:
                    await member.remove_roles(muted_role, reason="Mute expirado")
                except Exception:
                    pass
            bot.loop.create_task(unmute_later())
    except Exception as e:
        await ctx.send(f"❌ Error al silenciar: {e}")

@bot.command(name="unmute")
@is_staff_or_owner()
async def cmd_unmute(ctx, member: discord.Member):
    guild = ctx.guild
    muted_role = guild.get_role(int(MUTE_ROLE_ID)) if MUTE_ROLE_ID else discord.utils.get(guild.roles, name="Muted")
    if not muted_role:
        await ctx.send("No existe rol 'Muted'.")
        return
    try:
        await member.remove_roles(muted_role, reason="Desmute manual")
        await ctx.send(f"🔊 {member.mention} ha sido des-silenciado.")
        await dispatch_alert(guild, ctx.message, member, f"Usuario 
