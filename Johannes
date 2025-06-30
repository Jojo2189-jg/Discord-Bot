import threading
import discord
from discord.ext import commands
from discord import app_commands
import random
import asyncio
import sqlite3
import os
import datetime
import aiohttp
from dotenv import load_dotenv

# Lade Umgebungsvariablen aus der .env-Datei
load_dotenv()
TOKEN = os.getenv("DISCORD_BOT_TOKEN")
WELCOME_CHANNEL_ID = int(os.getenv("WELCOME_CHANNEL_ID", 0))
ONLINE_MESSAGE_CHANNEL_ID = int(os.getenv("ONLINE_MESSAGE_CHANNEL_ID", 0))
PING_ROLE_ID = int(os.getenv("PING_ROLE_ID", 0))
NEWSAPI_KEY = os.getenv("NEWSAPI_KEY")
POSTMORTEM_CHANNEL_ID = int(os.getenv("POSTMORTEM_CHANNEL_ID", 0))
MENTOR_ROLE_ID = int(os.getenv("MENTOR_ROLE_ID", 0))
TICKET_CHANNEL_ID = int(os.getenv("TICKET_CHANNEL_ID", 0))
TICKET_CATEGORY_ID = int(os.getenv("TICKET_CATEGORY_ID", 0))
MOD_ROLE_ID = int(os.getenv("MOD_ROLE_ID", 0)) # Füge diese Zeile hinzu, wenn du eine spezifische Mod-Rolle hast

# Verzeichnis für Trading-Bilder erstellen, falls es nicht existiert
TRADE_IMAGE_DIR = "trading_images"
os.makedirs(TRADE_IMAGE_DIR, exist_ok=True)

# Dictionary, um aktive Umfragen zu speichern
active_polls = {}
# Dictionary, um aktive Postmortems zu speichern
active_postmortems = {}
# Dictionary, um aktive Tickets zu speichern (für Inaktivitäts-Tracking und Zuordnung)
active_tickets = {} # Format: {channel_id: {'user_id': user_id, 'last_activity': datetime_obj, 'ask_more_questions_message_id': None, 'is_closing_timeout_active': False}}

# Discord Intents konfigurieren (welche Ereignisse der Bot empfangen soll)
intents = discord.Intents.default()
intents.members = True
intents.message_content = True
bot = commands.Bot(command_prefix="!", intents=intents)

# --- Datenbank-Helferfunktionen ---
def get_db_connection():
    """Stellt eine Verbindung zur SQLite-Datenbank her."""
    conn = sqlite3.connect('trading_journal.db')
    conn.execute('PRAGMA foreign_keys = ON;')
    conn.row_factory = sqlite3.Row
    return conn

async def create_tables():
    """Erstellt die 'trades', 'postmortems' und 'mentor_xp' Tabellen, falls sie noch nicht existieren."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS trades (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            asset TEXT NOT NULL,
            direction TEXT NOT NULL,
            entry_price REAL NOT NULL,
            exit_price REAL,
            pnl_amount REAL NOT NULL,
            pnl_percent REAL NOT NULL,
            comment TEXT,
            image_filename TEXT,
            timestamp TEXT NOT NULL
        );
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS postmortems (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            message_id INTEGER NOT NULL,
            asset TEXT NOT NULL,
            direction TEXT NOT NULL,
            pnl_amount REAL NOT NULL,
            pnl_percent REAL NOT NULL,
            reasoning TEXT NOT NULL,
            image_filename TEXT,
            timestamp TEXT NOT NULL
        );
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS mentor_xp (
            user_id INTEGER PRIMARY KEY,
            xp INTEGER DEFAULT 0
        );
    ''')
    conn.commit()
    conn.close()
    print("✅ Datenbanktabellen 'trades', 'postmortems' und 'mentor_xp' überprüft/erstellt.")

async def add_xp(user_id: int, xp_amount: int):
    """Fügt einem Benutzer XP hinzu oder aktualisiert diese."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute(
        """
        INSERT INTO mentor_xp (user_id, xp)
        VALUES (?, ?)
        ON CONFLICT(user_id) DO UPDATE SET
            xp = xp + EXCLUDED.xp;
        """,
        (user_id, xp_amount)
    )
    conn.commit()
    conn.close()
    print(f"Added {xp_amount} XP to user {user_id}")

def is_mentor(member: discord.Member) -> bool:
    """Prüft, ob ein Mitglied die Mentor-Rolle besitzt."""
    if MENTOR_ROLE_ID == 0:
        print("⚠️ MENTOR_ROLE_ID ist nicht in der .env konfiguriert. XP-System für Mentoren wird möglicherweise nicht korrekt funktionieren.")
        return False
    return any(role.id == MENTOR_ROLE_ID for role in member.roles)

def is_moderator(member: discord.Member) -> bool:
    """Prüft, ob ein Mitglied ein Moderator oder Administrator ist."""
    if member.guild_permissions.administrator:
        return True
    if MOD_ROLE_ID != 0:
        return any(role.id == MOD_ROLE_ID for role in member.roles)
    return False


# --- Begrüßungsnachrichten-Listen ---
DM_WELCOME_SAYINGS = [
    "👋 Herzlich willkommen auf dem Server! Schön, dass du unsere Trading-Community gefunden hast.",
    "📈 Bereit für dein Trading-Abenteuer? Denk dran: Verluste sind nur unrealisiert, wenn du nie verkaufst 😉",
    "💡 Die Börse ist ein Ort, an dem man Geld von den Ungeduldigen zu den Geduldigen transferiert. Bleib geduldig und lerne mit uns!",
    "✉️ Wenn du Fragen hast oder Hilfe brauchst, zögere nicht, die Admins oder Mods zu kontaktieren!",
    "🚀 Willkommen an Bord! Mögen deine Trades grün und deine Verluste klein sein.",
    "Hey, schön dich hier zu sehen! Wir sind gespannt auf deine Beiträge zur Community.",
    "Willkommen in der JCT Trading Community! Hier findest du Gleichgesinnte für deine Trading-Reise."
]

CHANNEL_WELCOME_SAYINGS_COMPACT = [
    "ist gelandet! Zeit für Action!",
    "erscheint! Vorsicht, Chart-Monster!",
    "hat den Weg gefunden! Willkommen!",
    "ist hier! Hoffentlich mit Gewinnabsicht. 😉",
    "stürzt sich ins Trading-Getümel!",
    "bringt frischen Wind! Willkommen!",
    "hat den Call des Lebens gemacht... hierher zu kommen! 😄",
    "ist da! Mögen die Gewinne mit dir sein!",
    "beginnt die nächste Trade-Reise!",
    "checkt ein! Bereit für den nächsten Pump?",
    "ist eingetroffen! Let's make some money! (Oder auch nicht. 🤷‍♂️)"
]

# --- Terminal-Interaktion (läuft in einem separaten Thread) ---
def run_terminal_interface(bot_instance, loop):
    print("\n-------------------------------------------")
    print("Bot Terminal-Modus. Befehle:")
    print("  list channels           - Zeigt alle Textkanäle und ihre IDs an")
    print("  send <kanal_id> <nachricht> - Sendet eine Nachricht an einen Kanal")
    print("  exit                    - Beendet den Terminal-Modus (Bot läuft weiter)")
    print("  shutdown                - Fährt den Bot komplett herunter")
    print("-------------------------------------------\n")

    while True:
        try:
            command = input("> ")
            if command.lower() == "exit":
                print("Terminal-Modus beendet. Bot läuft weiter.")
                break
            elif command.lower() == "shutdown":
                print("Bot wird heruntergefahren...")
                asyncio.run_coroutine_threadsafe(bot_instance.close(), loop)
                break
            elif command.lower() == "list channels":
                print("Verfügbare Textkanäle:")
                for guild in bot_instance.guilds:
                    print(f"  Server: {guild.name} (ID: {guild.id})")
                    for channel in guild.text_channels:
                        print(f"    - {channel.name} (ID: {channel.id})")
            elif command.lower().startswith("send "):
                parts = command.split(" ", 2)
                if len(parts) >= 3:
                    try:
                        channel_id = int(parts[1])
                        message_text = parts[2]
                        asyncio.run_coroutine_threadsafe(
                            bot_instance.get_channel(channel_id).send(message_text), loop
                        )
                        print(f"Nachricht an Kanal {channel_id} zur Ausführung übergeben.")
                    except ValueError:
                        print("Ungültige Kanal-ID. Bitte gib eine Zahl ein.")
                    except Exception as e:
                        print(f"Fehler beim Senden der Nachricht: {e}")
                else:
                    print("Ungültiger 'send' Befehl. Syntax: send <kanal_id> <nachricht>")
            else:
                print("Unbekannter Befehl.")
        except EOFError:
            print("\nEOF erkannt. Terminal-Modus wird beendet.")
            break
        except Exception as e:
            print(f"Fehler im Terminal-Modus: {e}")

# --- Bot-Ereignisse ---
@bot.event
async def on_ready():
    """Wird ausgeführt, wenn der Bot erfolgreich mit Discord verbunden ist."""
    print(f"✅ Bot ist online als {bot.user.name}#{bot.user.discriminator}")
    print(f"Bot-ID: {bot.user.id}")
    await create_tables()

    try:
        synced = await bot.tree.sync()
        print(f"✅ {len(synced)} Slash Commands synchronisiert.")
    except Exception as e:
        print(f"⚠️ Fehler beim Synchronisieren der Slash Commands: {e}")


    try:
        online_channel = bot.get_channel(ONLINE_MESSAGE_CHANNEL_ID)
        if online_channel:
            ping_mention = ""
            if PING_ROLE_ID != 0:
                guild = online_channel.guild
                role = guild.get_role(PING_ROLE_ID)
                if role:
                    ping_mention = f"{role.mention} "
                else:
                    print(f"⚠️ Warnung: Rolle mit ID {PING_ROLE_ID} auf Server '{guild.name}' nicht gefunden. Kann nicht pingen.")

            message_content = f"{ping_mention}Hallo liebe Community! Ich stehe euch jetzt zur Verfügung! 🚀"
            online_embed = discord.Embed(
                title="✅ Bot ist online!",
                description=message_content,
                color=discord.Color.green()
            )
            online_embed.set_footer(text="Bereit, euch in der Trading Community zu unterstützen!")
            await online_channel.send(embed=online_embed)
            print(f"📨 Online-Nachricht in #{online_channel.name} gesendet.")
        else:
            print(f"⚠️ Online-Nachrichten-Kanal mit ID {ONLINE_MESSAGE_CHANNEL_ID} nicht gefunden.")
    except Exception as e:
        print(f"⚠️ Fehler beim Senden der Online-Nachricht: {e}")

    # Füge die persistente View für den "Ticket öffnen"-Button hinzu
    bot.add_view(OpenTicketView())
    # Füge TicketCloseView mit einer Dummy-ID hinzu, damit der Bot sie registriert.
    # Sie wird dann mit der echten ID instanziiert, wenn ein Ticket erstellt wird.
    bot.add_view(TicketCloseView(0))
    # ConfirmCloseTicketView wird NICHT hier hinzugefügt, da sie einen Timeout hat
    # und nur temporär in den Ticket-Kanälen angezeigt wird.

    # Sende die Startnachricht mit dem Button, wenn noch nicht geschehen
    await send_ticket_open_message()

    # Starte den Hintergrund-Task für die Ticket-Inaktivität erst, wenn der Bot bereit ist
    bot.loop.create_task(check_ticket_inactivity())
    print("✅ Ticket-Inaktivitäts-Check gestartet.")

    loop = asyncio.get_event_loop()
    threading.Thread(target=run_terminal_interface, args=(bot, loop)).start()


@bot.event
async def on_member_join(member):
    """Wird ausgeführt, wenn ein neues Mitglied dem Server beitritt."""
    try:
        dm_message_content = random.choice(DM_WELCOME_SAYINGS)
        await member.send(
            f"👋 **Herzlich willkommen auf dem Server, {member.name}!**\n\n"
            f"{dm_message_content}\n\n"
            f"Schau dich gerne auf dem Server um!"
        )
    except discord.Forbidden:
        print(f"⚠️ Konnte keine DM an {member.name} senden. Möglicherweise haben sie DMs deaktiviert.")
    except Exception as e:
        print(f"⚠️ Fehler beim Senden der Willkommens-DM an {member.name}: {e}")

    try:
        welcome_channel = bot.get_channel(WELCOME_CHANNEL_ID)
        if welcome_channel:
            random_channel_saying = random.choice(CHANNEL_WELCOME_SAYINGS_COMPACT)
            channel_message = f"🟢➡️ **{member.name}** {random_channel_saying}"
            await welcome_channel.send(channel_message)
        else:
            print(f"⚠️ Willkommens-Channel mit ID {WELCOME_CHANNEL_ID} nicht gefunden.")
    except Exception as e:
        print(f"⚠️ Fehler beim Senden der Willkommensnachricht im Kanal: {e}")


@bot.event
async def on_message(message):
    """Wird ausgeführt, wenn eine Nachricht gesendet wird."""
    if message.author == bot.user:
        return # Ignoriere Nachrichten vom Bot selbst

    # Update last_activity for tickets
    if message.channel.id in active_tickets:
        ticket_data = active_tickets[message.channel.id]
        # Nur relevante Nachrichten aktualisieren die Aktivität
        if message.author.id == ticket_data['user_id'] or is_moderator(message.author):
            ticket_data['last_activity'] = datetime.datetime.now()
            # Falls ein "Noch Fragen?" Timer läuft, setze ihn zurück (bzw. lösche die zugehörige Nachricht)
            if ticket_data['is_closing_timeout_active']:
                # Wenn der User oder ein Mod antwortet, während der "Noch Fragen?"-Timeout aktiv ist
                # Deaktiviere den Timeout und lösche die "Noch Fragen?"-Nachricht
                ticket_data['is_closing_timeout_active'] = False
                if ticket_data['ask_more_questions_message_id']:
                    try:
                        msg_to_delete = await message.channel.fetch_message(ticket_data['ask_more_questions_message_id'])
                        await msg_to_delete.delete()
                        ticket_data['ask_more_questions_message_id'] = None
                        print(f"Zurückgesetzter 'Noch Fragen?' Timer für Ticket {message.channel.id} durch neue Nachricht.")
                    except discord.NotFound:
                        ticket_data['ask_more_questions_message_id'] = None # Nachricht wurde bereits gelöscht oder existierte nicht
                    except Exception as e:
                        print(f"Fehler beim Löschen der 'Noch Fragen?'-Nachricht: {e}")

    await bot.process_commands(message) # Wichtig, damit normale Befehle auch funktionieren


# --- Ticketsystem ---

class OpenTicketView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None) # Timeout=None macht die View persistent

    @discord.ui.button(label="Ticket öffnen", style=discord.ButtonStyle.green, custom_id="open_ticket_button", emoji="📩")
    async def open_ticket_button_callback(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.defer(ephemeral=True)

        guild = interaction.guild
        member = interaction.user

        if TICKET_CATEGORY_ID == 0:
            await interaction.followup.send("Fehler: Ticket-Kategorie-ID ist im Bot nicht konfiguriert.", ephemeral=True)
            return

        category = guild.get_channel(TICKET_CATEGORY_ID)
        if not category or not isinstance(category, discord.CategoryChannel):
            await interaction.followup.send("Fehler: Konfigurierte Ticket-Kategorie nicht gefunden oder ist keine Kategorie. Bitte kontaktiere einen Administrator.", ephemeral=True)
            print(f"ERROR: Ticket category with ID {TICKET_CATEGORY_ID} not found or not a category.")
            return

        # Prüfen, ob der Benutzer bereits ein offenes Ticket hat
        for channel_id, ticket_data in list(active_tickets.items()): # list() zum Iterieren über eine Kopie
            if ticket_data['user_id'] == member.id:
                try:
                    existing_channel = guild.get_channel(channel_id)
                    # Überprüfe, ob der Kanal noch existiert und sich in der richtigen Kategorie befindet
                    if existing_channel and existing_channel.category_id == TICKET_CATEGORY_ID:
                        await interaction.followup.send(f"Du hast bereits ein offenes Ticket: {existing_channel.mention}", ephemeral=True)
                        return
                    else: # Kanal existiert nicht mehr oder ist nicht in der richtigen Kategorie, aufräumen
                        print(f"WARNUNG: Ticket {channel_id} für Benutzer {member.id} war in active_tickets, aber Kanal nicht gefunden/falsche Kategorie. Wird entfernt.")
                        del active_tickets[channel_id]
                except Exception as e:
                    print(f"Fehler beim Prüfen auf bestehendes Ticket: {e}. Ticket {channel_id} wird entfernt.")
                    del active_tickets[channel_id] # Aufräumen bei Fehler


        # Kanalname
        ticket_name = f"ticket-{member.name.lower().replace(' ', '-')}-{random.randint(1000, 9999)}"
        ticket_name = ''.join(c for c in ticket_name if c.isalnum() or c == '-')

        # Berechtigungen für den neuen Kanal
        overwrites = {
            guild.default_role: discord.PermissionOverwrite(read_messages=False),
            member: discord.PermissionOverwrite(read_messages=True, send_messages=True, embed_links=True, attach_files=True),
            guild.me: discord.PermissionOverwrite(read_messages=True, send_messages=True, embed_links=True, attach_files=True),
        }

        # Füge Moderatoren und Administratoren hinzu
        mod_role = guild.get_role(MOD_ROLE_ID) if MOD_ROLE_ID != 0 else None
        if mod_role:
            overwrites[mod_role] = discord.PermissionOverwrite(read_messages=True, send_messages=True, embed_links=True, attach_files=True)
        else:
            print("⚠️ MOD_ROLE_ID ist nicht konfiguriert. Nur Admins können Tickets standardmäßig sehen.")
            # Füge alle Rollen mit Administrator-Berechtigungen hinzu, wenn keine spezifische Mod-Rolle definiert ist
            for role in guild.roles:
                if role.permissions.administrator:
                    overwrites[role] = discord.PermissionOverwrite(read_messages=True, send_messages=True, embed_links=True, attach_files=True)


        try:
            ticket_channel = await guild.create_text_channel(ticket_name, category=category, overwrites=overwrites)
            await interaction.followup.send(f"Dein Ticket wurde erstellt: {ticket_channel.mention}", ephemeral=True)

            welcome_embed = discord.Embed(
                title="📩 Dein Ticket wurde geöffnet!",
                description=f"Hallo {member.mention},\n\n"
                            "Bitte beschreibe dein Anliegen so detailliert wie möglich. Ein Moderator wird sich bald bei dir melden.",
                color=discord.Color.blue()
            )
            welcome_embed.set_footer(text="Dieses Ticket wird bei Inaktivität automatisch geschlossen.")

            # Erstelle die TicketCloseView mit der korrekten Channel-ID
            ticket_close_view = TicketCloseView(ticket_channel.id)
            await ticket_channel.send(embed=welcome_embed, view=ticket_close_view)

            active_tickets[ticket_channel.id] = {
                'user_id': member.id,
                'last_activity': datetime.datetime.now(),
                'ask_more_questions_message_id': None,
                'is_closing_timeout_active': False # Flag for 3-hour inactivity timeout
            }
            print(f"Ticket {ticket_channel.name} (ID: {ticket_channel.id}) opened by {member.name} (ID: {member.id})")

        except discord.Forbidden:
            await interaction.followup.send("Ich habe keine Berechtigung, Kanäle zu erstellen. Bitte kontaktiere einen Administrator.", ephemeral=True)
            print("ERROR: Bot lacks permissions to create channels.")
        except Exception as e:
            await interaction.followup.send(f"Ein Fehler ist aufgetreten: {e}", ephemeral=True)
            print(f"ERROR creating ticket: {e}")

async def send_ticket_open_message():
    """Sendet die initiale Nachricht mit dem 'Ticket öffnen'-Button in den dedizierten Kanal."""
    await bot.wait_until_ready() # Stelle sicher, dass der Bot bereit ist

    if TICKET_CHANNEL_ID == 0:
        print("⚠️ TICKET_CHANNEL_ID ist nicht in der .env konfiguriert. Der 'Ticket öffnen'-Button wird nicht gepostet.")
        return

    ticket_channel = bot.get_channel(TICKET_CHANNEL_ID)
    if not ticket_channel:
        print(f"⚠️ Ticket-Kanal mit ID {TICKET_CHANNEL_ID} nicht gefunden. Der 'Ticket öffnen'-Button wird nicht gepostet.")
        return

    # Prüfen, ob die Nachricht bereits existiert, um Doppelposts zu vermeiden
    # Suchen wir nach einer Nachricht des Bots, die den Ticket-Button enthält
    try:
        async for message in ticket_channel.history(limit=50):
            if message.author == bot.user and message.embeds:
                if "Klicke auf den Button unten" in message.embeds[0].description:
                    print("Initialer Ticket-Button scheint bereits im Kanal zu sein.")
                    # Verknüpfe die View erneut, falls der Bot neu gestartet wurde und sie verloren ging
                    await message.edit(view=OpenTicketView())
                    return # Beende die Funktion, da der Button bereits existiert
    except discord.Forbidden:
        print(f"⚠️ Bot hat keine Berechtigung, im Kanal {ticket_channel.name} den Verlauf zu lesen.")
        # Wenn wir den Verlauf nicht lesen können, versuchen wir einfach, die Nachricht zu senden
    except Exception as e:
        print(f"Fehler beim Prüfen auf vorhandene Ticket-Nachricht: {e}")

    # Wenn kein Button gefunden wurde, sende einen neuen
    embed = discord.Embed(
        title="📩 Support Ticket",
        description="Klicke auf den Button unten, um ein neues Support-Ticket zu öffnen. Ein privater Kanal wird für dich erstellt.",
        color=discord.Color.blurple()
    )
    embed.set_footer(text="Bitte öffne nur ein Ticket pro Anliegen.")

    await ticket_channel.send(embed=embed, view=OpenTicketView())
    print(f"Initialer Ticket-Button im Kanal #{ticket_channel.name} gepostet.")


class TicketCloseView(discord.ui.View):
    def __init__(self, ticket_channel_id):
        super().__init__(timeout=None) # Timeout=None für Persistenz
        self.ticket_channel_id = ticket_channel_id

    @discord.ui.button(label="Ticket schließen", style=discord.ButtonStyle.red, custom_id="close_ticket_button", emoji="🔒")
    async def close_ticket_button_callback(self, interaction: discord.Interaction, button: discord.ui.Button):
        channel = interaction.channel
        if channel.id != self.ticket_channel_id:
            await interaction.response.send_message("Dieser Button ist für ein anderes Ticket gedacht.", ephemeral=True)
            return

        is_mod_or_admin = is_moderator(interaction.user)
        ticket_data = active_tickets.get(channel.id, {})
        ticket_owner_id = ticket_data.get('user_id')

        # Prüfe, ob der Interagierende der Ticket-Besitzer oder ein Mod/Admin ist
        if is_mod_or_admin or interaction.user.id == ticket_owner_id:
            # Lösche die vorherige "Noch Fragen?"-Nachricht, falls vorhanden, wenn ein neuer Schließversuch initiiert wird
            if ticket_data.get('ask_more_questions_message_id'):
                try:
                    msg_to_delete = await channel.fetch_message(ticket_data['ask_more_questions_message_id'])
                    await msg_to_delete.delete()
                    ticket_data['ask_more_questions_message_id'] = None
                    ticket_data['is_closing_timeout_active'] = False
                except discord.NotFound:
                    pass # Nachricht schon weg
                except Exception as e:
                    print(f"Fehler beim Löschen der alten 'Noch Fragen?'-Nachricht: {e}")

            if is_mod_or_admin and interaction.user.id != ticket_owner_id:
                # MOD initiiert Schließung: Frage "Noch Fragen?" mit Option zum sofortigen Schließen
                confirm_view = ConfirmCloseTicketView(channel.id, is_mod_initiated=True)
                # Speichere die gesendete Nachricht, um sie später bei Interaktion oder Timeout zu löschen
                await interaction.response.send_message(
                    "Möchtest du dieses Ticket schließen? Noch Fragen? "
                    "Wenn der Benutzer nicht mehr antwortet, kannst du es hier sofort schließen.",
                    view=confirm_view
                )
                response_message = await interaction.original_response()
                # Da diese View jetzt temporär ist, muss die Nachricht auch zur View hinzugefügt werden
                confirm_view.message = response_message # Speichere die Nachricht in der View-Instanz
                active_tickets[channel.id]['ask_more_questions_message_id'] = response_message.id
                active_tickets[channel.id]['is_closing_timeout_active'] = True # Setze Flag für den Timeout, falls keine Antwort kommt
                active_tickets[channel.id]['last_activity'] = datetime.datetime.now() # Reset timer
            else: # Nutzer oder Mod/Admin, der sein eigenes Ticket schließt
                # User (oder Mod für eigenes Ticket) schließt: Direkte Bestätigung
                confirm_view = ConfirmCloseTicketView(channel.id, is_user=True)
                await interaction.response.send_message(
                    "Bist du sicher, dass du dein Ticket schließen möchtest?",
                    view=confirm_view
                )
                response_message = await interaction.original_response()
                confirm_view.message = response_message # Speichere die Nachricht in der View-Instanz
        else:
            await interaction.response.send_message("Du hast keine Berechtigung, dieses Ticket zu schließen.", ephemeral=True)


class ConfirmCloseTicketView(discord.ui.View):
    def __init__(self, ticket_channel_id, is_user=False, is_mod_initiated=False):
        super().__init__(timeout=180) # Timeout für die Ja/Nein-Antwort (3 Minuten). Diese View ist nicht persistent im Sinne von bot.add_view.
        self.ticket_channel_id = ticket_channel_id
        self.is_user = is_user
        self.is_mod_initiated = is_mod_initiated
        self.message = None # Wird gesetzt, nachdem die Nachricht gesendet wurde

        # Füge Buttons hinzu, basierend auf dem Kontext
        if is_mod_initiated:
            # Mod-initiierte "Noch Fragen?"-Ansicht
            btn_yes_questions = discord.ui.Button(label="Ja, ich habe Fragen", style=discord.ButtonStyle.green, custom_id="mod_confirm_yes_questions")
            btn_yes_questions.callback = self.confirm_yes_callback # Callback zuweisen
            self.add_item(btn_yes_questions)

            btn_no_questions = discord.ui.Button(label="Nein, alles klar", style=discord.ButtonStyle.red, custom_id="mod_confirm_no_questions")
            btn_no_questions.callback = self.confirm_no_callback # Callback zuweisen
            self.add_item(btn_no_questions)

            btn_force_close = discord.ui.Button(label="Ticket jetzt schließen (Mod)", style=discord.ButtonStyle.grey, custom_id="mod_force_close_ticket")
            btn_force_close.callback = self.force_close_ticket_callback # Callback zuweisen
            self.add_item(btn_force_close)
        elif is_user:
            # Benutzer-initiierte Bestätigungs-Ansicht
            btn_yes_user = discord.ui.Button(label="Ja, Ticket schließen", style=discord.ButtonStyle.green, custom_id="user_confirm_close_yes")
            btn_yes_user.callback = self.confirm_close_yes_user_callback # Callback zuweisen
            self.add_item(btn_yes_user)

            btn_no_user = discord.ui.Button(label="Nein, doch nicht", style=discord.ButtonStyle.red, custom_id="user_confirm_close_no")
            btn_no_user.callback = self.confirm_close_no_user_callback # Callback zuweisen
            self.add_item(btn_no_user)


    async def on_timeout(self):
        # Wenn der Benutzer nicht innerhalb der Bestätigungs-Timeout-Zeit antwortet
        channel = bot.get_channel(self.ticket_channel_id)
        if channel and self.is_mod_initiated and active_tickets.get(channel.id, {}).get('is_closing_timeout_active'):
            # Hier löschen wir nur die "Noch Fragen?"-Nachricht. Der 3-Stunden-Timeout wird dann das Ticket schließen,
            # wenn der User weiterhin inaktiv bleibt.
            if active_tickets[channel.id]['ask_more_questions_message_id']:
                try:
                    msg_to_delete = await channel.fetch_message(active_tickets[channel.id]['ask_more_questions_message_id'])
                    await msg_to_delete.delete()
                    active_tickets[channel.id]['ask_more_questions_message_id'] = None
                except discord.NotFound:
                    pass
                except Exception as e:
                    print(f"Fehler beim Löschen der 'Noch Fragen?'-Nachricht im Timeout: {e}")
            print(f"Bestätigungs-View für Ticket {channel.id} ist abgelaufen. Nutzer hat nicht geantwortet auf Mod-Nachricht.")
        elif channel and self.is_user:
            # Wenn der Benutzer seine eigene Schließungsanfrage nicht bestätigt
            await channel.send("Deine Anfrage zum Schließen des Tickets ist abgelaufen. Das Ticket bleibt offen.")
            if active_tickets.get(channel.id):
                active_tickets[channel.id]['is_closing_timeout_active'] = False
                active_tickets[channel.id]['last_activity'] = datetime.datetime.now()

        for item in self.children: # Deaktiviere alle Buttons in dieser View, wenn der Timeout erreicht ist
            item.disabled = True
        if self.message: # Aktualisiere die Nachricht, um Buttons zu deaktivieren
            try:
                await self.message.edit(view=self)
            except discord.HTTPException:
                pass # Nachricht könnte bereits gelöscht sein

        self.stop() # Beende diese View

    # --- Buttons für die Mod-initiierte Abfrage ("Noch Fragen?") ---
    # KEIN @discord.ui.button DECORATOR HIER!
    async def confirm_yes_callback(self, interaction: discord.Interaction):
        await interaction.response.defer() # Defer die Antwort, um Timeout zu vermeiden
        channel = interaction.channel
        if channel.id != self.ticket_channel_id:
            await interaction.followup.send("Diese Antwort ist für ein anderes Ticket gedacht.", ephemeral=True)
            return

        # Lösche die vorherige "Noch Fragen?"-Nachricht
        if channel.id in active_tickets and active_tickets[channel.id]['ask_more_questions_message_id']:
            try:
                msg_id = active_tickets[channel.id]['ask_more_questions_message_id']
                msg_to_delete = await channel.fetch_message(msg_id)
                await msg_to_delete.delete()
            except discord.NotFound:
                pass
            except Exception as e:
                print(f"Fehler beim Löschen der 'Noch Fragen?'-Nachricht: {e}")
            active_tickets[channel.id]['ask_more_questions_message_id'] = None
            active_tickets[channel.id]['is_closing_timeout_active'] = False # Timeout deaktivieren

        await interaction.followup.send("Alles klar. Bitte stelle deine Fragen.")
        active_tickets[channel.id]['last_activity'] = datetime.datetime.now() # Aktivität aktualisieren
        self.stop() # Beende diese View

    # KEIN @discord.ui.button DECORATOR HIER!
    async def confirm_no_callback(self, interaction: discord.Interaction):
        await interaction.response.defer()
        channel = interaction.channel
        if channel.id != self.ticket_channel_id:
            await interaction.followup.send("Diese Antwort ist für ein anderes Ticket gedacht.", ephemeral=True)
            return

        # Lösche die vorherige "Noch Fragen?"-Nachricht
        if channel.id in active_tickets and active_tickets[channel.id]['ask_more_questions_message_id']:
            try:
                msg_id = active_tickets[channel.id]['ask_more_questions_message_id']
                msg_to_delete = await channel.fetch_message(msg_id)
                await msg_to_delete.delete()
            except discord.NotFound:
                pass
            except Exception as e:
                print(f"Fehler beim Löschen der 'Noch Fragen?'-Nachricht: {e}")
            active_tickets[channel.id]['ask_more_questions_message_id'] = None

        await interaction.followup.send("Alles klar. Ticket wird geschlossen.")
        await self.close_ticket(channel, reason="User confirmed no more questions.")
        self.stop()

    # KEIN @discord.ui.button DECORATOR HIER!
    async def force_close_ticket_callback(self, interaction: discord.Interaction):
        await interaction.response.defer()
        channel = interaction.channel
        if channel.id != self.ticket_channel_id:
            await interaction.followup.send("Dieser Button ist für ein anderes Ticket gedacht.", ephemeral=True)
            return

        if not is_moderator(interaction.user):
            await interaction.followup.send("Du hast keine Berechtigung, dieses Ticket manuell zu schließen.", ephemeral=True)
            return

        # Lösche die vorherige "Noch Fragen?"-Nachricht, falls vorhanden
        if channel.id in active_tickets and active_tickets[channel.id]['ask_more_questions_message_id']:
            try:
                msg_id = active_tickets[channel.id]['ask_more_questions_message_id']
                msg_to_delete = await channel.fetch_message(msg_id)
                await msg_to_delete.delete()
            except discord.NotFound:
                pass
            except Exception as e:
                print(f"Fehler beim Löschen der 'Noch Fragen?'-Nachricht: {e}")
            active_tickets[channel.id]['ask_more_questions_message_id'] = None
            active_tickets[channel.id]['is_closing_timeout_active'] = False

        await interaction.followup.send("Ticket wird auf Anweisung des Moderators geschlossen...")
        await self.close_ticket(channel, reason=f"Manuell geschlossen von {interaction.user.display_name} (Mod).")
        self.stop()

    # --- Buttons für die Benutzer-initiierte Bestätigung ---
    # KEIN @discord.ui.button DECORATOR HIER!
    async def confirm_close_yes_user_callback(self, interaction: discord.Interaction):
        await interaction.response.defer()
        channel = interaction.channel
        if channel.id != self.ticket_channel_id:
            await interaction.followup.send("Diese Antwort ist für ein anderes Ticket gedacht.", ephemeral=True)
            return
        await interaction.followup.send("Ticket wird geschlossen...")
        await self.close_ticket(channel, reason="User confirmed closure.")
        self.stop()

    # KEIN @discord.ui.button DECORATOR HIER!
    async def confirm_close_no_user_callback(self, interaction: discord.Interaction):
        await interaction.response.defer()
        channel = interaction.channel
        if channel.id != self.ticket_channel_id:
            await interaction.followup.send("Diese Antwort ist für ein anderes Ticket gedacht.", ephemeral=True)
            return
        await interaction.followup.send("Das Ticket bleibt offen.")
        active_tickets[channel.id]['is_closing_timeout_active'] = False # Timeout deaktivieren
        active_tickets[channel.id]['last_activity'] = datetime.datetime.now() # Aktivität aktualisieren
        self.stop()

    async def close_ticket(self, channel, timeout_close=False, reason="Ticket closed."):
        try:
            if channel:
                # Sicherstellen, dass das Ticket im active_tickets Dictionary ist, bevor wir es entfernen
                if channel.id in active_tickets:
                    del active_tickets[channel.id]

                embed = discord.Embed(
                    title="Ticket geschlossen",
                    description=f"Dieses Ticket von {channel.mention} wurde geschlossen.",
                    color=discord.Color.dark_red()
                )
                if timeout_close:
                    embed.description += "\nEs wurde aufgrund von Inaktivität automatisch geschlossen."
                embed.add_field(name="Grund des Schließens", value=reason, inline=False)


                # Optional: Sende eine Nachricht an einen Logging-Kanal, wenn vorhanden
                # log_channel = bot.get_channel(YOUR_LOG_CHANNEL_ID) # Ersetze YOUR_LOG_CHANNEL_ID
                # if log_channel:
                #     await log_channel.send(embed=embed)

                await channel.delete()
                print(f"Ticket {channel.name} (ID: {channel.id}) closed. Reason: {reason}")
            else:
                print(f"Versuch, nicht existierendes Ticket {self.ticket_channel_id} zu schließen.")
        except discord.Forbidden:
            print(f"FEHLER: Bot hat keine Berechtigung, Kanal {channel.name} zu löschen.")
        except Exception as e:
            print(f"Unerwarteter Fehler beim Schließen des Tickets {channel.id}: {e}")

# --- Background Task für Ticket-Inaktivität ---
async def check_ticket_inactivity():
    """Hintergrund-Task, der periodisch offene Tickets auf Inaktivität prüft."""
    await bot.wait_until_ready()
    while not bot.is_closed():
        await asyncio.sleep(60 * 5) # Alle 5 Minuten prüfen

        tickets_to_process = []
        now = datetime.datetime.now()

        # Erstelle eine Liste der Schlüssel, um sicher zu iterieren, während das Original-Dictionary geändert wird
        for channel_id, ticket_data in list(active_tickets.items()):
            last_activity = ticket_data['last_activity']
            time_diff = now - last_activity

            channel = bot.get_channel(channel_id)
            if not channel:
                print(f"Warnung: Ticket {channel_id} in active_tickets, aber Kanal nicht gefunden. Wird entfernt.")
                del active_tickets[channel_id] # Aufräumen
                continue

            # PHASE 1: Nach 3 Stunden Inaktivität die "Noch Fragen?"-Nachricht senden
            # Dies geschieht nur, wenn der Timeout noch NICHT aktiv ist
            if not ticket_data['is_closing_timeout_active'] and time_diff.total_seconds() > (3 * 3600):
                if ticket_data['ask_more_questions_message_id'] is None: # Nur einmal senden
                    try:
                        ask_embed = discord.Embed(
                            title="Noch Fragen zu deinem Ticket?",
                            description="Dieses Ticket war länger inaktiv. Wenn du keine weiteren Fragen hast oder dein Anliegen gelöst ist, wird das Ticket in 3 Stunden automatisch geschlossen.",
                            color=discord.Color.orange()
                        )
                        ask_embed.set_footer(text="Antworte in diesem Kanal oder nutze die Buttons.")

                        # Diese ConfirmCloseTicketView ist nur für den Mod-Fall der Bestätigung, sie wird nicht persistent registriert.
                        confirm_view = ConfirmCloseTicketView(channel.id, is_mod_initiated=True) # Mod-Version der Bestätigung
                        msg = await channel.send(f"{channel.guild.get_member(ticket_data['user_id']).mention}", embed=ask_embed, view=confirm_view)
                        confirm_view.message = msg # Verknüpfe die Nachricht mit der View
                        ticket_data['ask_more_questions_message_id'] = msg.id
                        ticket_data['is_closing_timeout_active'] = True # Setze Flag, dass der 3-Stunden-Timeout für die Antwort läuft
                        ticket_data['last_activity'] = datetime.datetime.now() # Setze Aktivität auf jetzt, um 3h Countdown für Antwort zu starten
                        print(f"Gesendet 'Noch Fragen?'-Prompt für Ticket {channel.name} (ID: {channel_id}).")
                    except Exception as e:
                        print(f"Fehler beim Senden der 'Noch Fragen?'-Nachricht für Ticket {channel.id}: {e}")
                        # Wenn Fehler beim Senden, setze Flag zurück, um es erneut zu versuchen
                        ticket_data['is_closing_timeout_active'] = False
                        ticket_data['ask_more_questions_message_id'] = None
            # PHASE 2: Wenn der "Noch Fragen?"-Timeout aktiv ist, nach weiteren 3 Stunden schließen
            elif ticket_data['is_closing_timeout_active'] and time_diff.total_seconds() > (3 * 3600):
                tickets_to_process.append(channel) # Füge Kanal zur Schließliste hinzu

        for channel in tickets_to_process:
            confirm_view = ConfirmCloseTicketView(channel.id) # Dummy-Instanz für close_ticket Aufruf
            await confirm_view.close_ticket(channel, timeout_close=True, reason="Automatically closed due to 3 hours of inactivity after 'More questions?' prompt.")


# --- Umfragesystem ---
class PollOptionsView(discord.ui.View):
    """Eine View-Klasse für die Interaktion mit Umfragen (Buttons)."""
    def __init__(self, options_dict, question_text):
        super().__init__(timeout=None)
        self.options_dict = options_dict
        self.poll_message_id = 0
        self.question_text = question_text

        for key, label in options_dict.items():
            button = discord.ui.Button(label=label, style=discord.ButtonStyle.primary, custom_id=f"poll_option_{key}")
            button.callback = self.poll_button_callback
            self.add_item(button)

        # Füge den "Ergebnisse anzeigen" Button hinzu
        show_results_btn = discord.ui.Button(label="Ergebnisse anzeigen", style=discord.ButtonStyle.grey, custom_id="poll_show_results_button")
        show_results_btn.callback = self.show_results_button_callback # Korrigierter Callback-Name
        self.add_item(show_results_btn)

    async def poll_button_callback(self, interaction: discord.Interaction):
        """Callback für jeden Umfrage-Button."""
        await interaction.response.defer(ephemeral=True) # Hinzugefügt für sofortige Reaktion

        # Überprüfe, ob custom_id überhaupt existiert, bevor darauf zugegriffen wird
        if interaction.data and 'custom_id' in interaction.data:
            option_key = interaction.data['custom_id'].split('_')[-1]
        else:
            await interaction.followup.send("Ein unerwarteter Fehler ist aufgetreten. Die Button-ID konnte nicht gelesen werden.", ephemeral=True)
            return

        user_id = interaction.user.id

        if self.poll_message_id == 0:
            self.poll_message_id = interaction.message.id

        if self.poll_message_id not in active_polls:
            await interaction.followup.send("Diese Umfrage ist nicht mehr aktiv.", ephemeral=True)
            return

        poll_data = active_polls[self.poll_message_id]

        if user_id in poll_data['votes'] and poll_data['votes'][user_id] != option_key:
            old_option_key = poll_data['votes'][user_id]
            old_option_label = poll_data['options'].get(old_option_key, 'Unbekannt')
            poll_data['votes'][user_id] = option_key
            await interaction.followup.send(f"Deine Stimme wurde von '{old_option_label}' zu '{self.options_dict[option_key]}' geändert.", ephemeral=True)
        elif user_id not in poll_data['votes']:
            poll_data['votes'][user_id] = option_key
            await interaction.followup.send(f"Du hast für '{self.options_dict[option_key]}' abgestimmt.", ephemeral=True)
        else:
            await interaction.followup.send(f"Du hast bereits für '{self.options_dict[option_key]}' abgestimmt.", ephemeral=True)


    # @discord.ui.button ist jetzt direkt in __init__ integriert für den "Ergebnisse anzeigen" Button.
    # Die Funktion bleibt aber als Methode bestehen.
    async def show_results_button_callback(self, interaction: discord.Interaction): # Callback-Name korrigiert
        """Button-Callback zum Anzeigen der Ergebnisse und Beenden der Umfrage."""
        await interaction.response.defer() # Hinzugefügt für sofortige Reaktion

        if self.poll_message_id == 0:
            self.poll_message_id = interaction.message.id

        if self.poll_message_id in active_polls:
            poll_data = active_polls[self.poll_message_id]

            is_creator = interaction.user.id == poll_data['creator']
            is_admin_or_mod = is_moderator(interaction.user) # Prüfe mit neuer is_moderator Funktion

            if not (is_creator or is_admin_or_mod):
                await interaction.followup.send("Nur der Ersteller oder ein Administrator/Moderator kann die Umfrage beenden.", ephemeral=True)
                return

            await self.end_poll_and_show_results(interaction.message, interaction)
        else:
            await interaction.followup.send("Diese Umfrage ist nicht mehr aktiv.", ephemeral=True)

    async def end_poll_and_show_results(self, message, interaction=None):
        """Beendet die Umfrage, zeigt Ergebnisse an und deaktiviert Buttons."""
        if message.id not in active_polls:
            if interaction:
                await interaction.response.send_message("Diese Umfrage ist nicht mehr aktiv.", ephemeral=True)
            return

        poll_data = active_polls.pop(message.id)
        votes = poll_data['votes']

        results = {option_key: 0 for option_key in self.options_dict.keys()}
        for option_key in votes.values():
            results[option_key] += 1

        total_votes = sum(results.values())

        results_text = ""
        for option_key, count in results.items():
            option_label = self.options_dict[option_key]
            percentage = (count / total_votes * 100) if total_votes > 0 else 0
            results_text += f"**{option_label}**: {count} Stimmen ({percentage:.2f}%)\n"

        result_embed = discord.Embed(
            title=f"📊 Ergebnisse der Umfrage: {self.question_text}",
            description=results_text,
            color=discord.Color.blue()
        )
        result_embed.set_footer(text=f"Insgesamt {total_votes} Stimmen.")

        for item in self.children:
            item.disabled = True

        await message.edit(embed=result_embed, view=self)
        if interaction:
            try:
                await interaction.followup.send("Umfrage beendet und Ergebnisse angezeigt!")
            except discord.HTTPException:
                pass

@bot.tree.command(name="umfrage", description="Erstelle eine interaktive Umfrage mit bis zu 5 Optionen.")
@app_commands.describe(
    question="Die Frage deiner Umfrage",
    option1="Erste Abstimmungsoption",
    option2="Zweite Abstimmungsoption",
    option3="Dritte Abstimmungsoption (optional)",
    option4="Vierte Abstimmungsoption (optional)",
    option5="Fünfte Abstimmungsoption (optional)",
    duration_minutes="Dauer der Umfrage in Minuten (Standard: 5 Minuten für Tests)" # Standard geändert
)
async def umfrage_command(
    interaction: discord.Interaction,
    question: str,
    option1: str,
    option2: str,
    option3: str = None,
    option4: str = None,
    option5: str = None,
    duration_minutes: int = 5 # Standard geändert
):
    """Slash-Befehl zum Erstellen einer Umfrage."""
    options = [option1, option2]
    if option3: options.append(option3)
    if option4: options.append(option4)
    if option5: options.append(option5)

    if len(options) < 2:
        await interaction.response.send_message("Du benötigst mindestens zwei Optionen für eine Umfrage.", ephemeral=True)
        return

    options_dict = {str(i+1): opt for i, opt in enumerate(options)}

    poll_embed = discord.Embed(
        title=f"📊 Umfrage: {question}",
        description="Stimme ab, indem du auf den entsprechenden Button klickst!",
        color=discord.Color.purple()
    )

    for key, value in options_dict.items():
        poll_embed.add_field(name=f"Option {key}", value=value, inline=False)

    poll_embed.set_footer(text=f"Umfrage erstellt von {interaction.user.display_name} | Dauer: {duration_minutes} Minuten")

    view = PollOptionsView(options_dict, question)
    await interaction.response.send_message(embed=poll_embed, view=view)

    poll_message = await interaction.original_response()

    active_polls[poll_message.id] = {
        'question': question,
        'options': options_dict,
        'votes': {},
        'creator': interaction.user.id
    }

    view.poll_message_id = poll_message.id
    view.message = poll_message

    if is_mentor(interaction.user):
        await add_xp(interaction.user.id, 15)

    await asyncio.sleep(duration_minutes * 60)

    try:
        final_message = await poll_message.channel.fetch_message(poll_message.id)
        await view.end_poll_and_show_results(final_message)
    except discord.NotFound:
        print(f"Umfrage-Nachricht {poll_message.id} nicht gefunden. Möglicherweise gelöscht.")
    except Exception as e:
        print(f"Fehler beim Beenden der Umfrage {poll_message.id}: {e}")

# --- Trading-Journal Funktionen ---
@bot.tree.command(name="logtrade", description="Logge einen abgeschlossenen Trade in dein Journal.")
@app_commands.describe(
    asset="Das gehandelte Asset (z.B. BTC, ETH, SPY)",
    direction="Die Richtung des Trades (Long/Short)",
    entry_price="Dein Einstiegspreis",
    exit_price="Dein Ausstiegspreis",
    pnl_amount="Dein Gewinn/Verlust in Währung",
    pnl_percent="Dein Gewinn/Verlust in Prozent",
    comment="Optionale Notizen zu diesem Trade",
    image_file="Ein optionales Bild deines Trades (Anhang)"
)
@app_commands.choices(
    direction=[
        app_commands.Choice(name="Long", value="Long"),
        app_commands.Choice(name="Short", value="Short"),
    ]
)
async def log_trade(
    interaction: discord.Interaction,
    asset: str,
    direction: str,
    entry_price: float,
    exit_price: float,
    pnl_amount: float,
    pnl_percent: float,
    comment: str = None,
    image_file: discord.Attachment = None
):
    """Slash-Befehl zum Loggen eines Trades mit optionalem Bild."""
    try:
        image_filename = None
        if image_file:
            timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            safe_asset = ''.join(char for char in asset if char.isalnum())
            image_filename = f"{interaction.user.id}_{safe_asset}_{timestamp}.png"
            image_path = os.path.join(TRADE_IMAGE_DIR, image_filename)
            await image_file.save(image_path)
            print(f"Bild gespeichert: {image_path}")

        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            """
            INSERT INTO trades (user_id, asset, direction, entry_price, exit_price, pnl_amount, pnl_percent, comment, image_filename, timestamp)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            """,
            (
                interaction.user.id,
                asset,
                direction,
                entry_price,
                exit_price,
                pnl_amount,
                pnl_percent,
                comment,
                image_filename,
                datetime.datetime.now().isoformat()
            )
        )
        conn.commit()
        conn.close()

        embed = discord.Embed(
            title="📈 Trade erfolgreich geloggt!",
            description=f"Dein Trade auf **{asset}** wurde gespeichert.",
            color=discord.Color.green()
        )
        embed.add_field(name="Richtung", value=direction, inline=True)
        embed.add_field(name="Einstieg", value=f"${entry_price:,.2f}", inline=True)
        embed.add_field(name="Ausstieg", value=f"${exit_price:,.2f}", inline=True)
        embed.add_field(name="P&L", value=f"${pnl_amount:,.2f} ({pnl_percent:.2f}%)", inline=False)
        if comment:
            embed.add_field(name="Kommentar", value=comment, inline=False)
        if image_filename:
            embed.add_field(name="Bild", value="Datei wurde gespeichert (lokal)", inline=False)

        await interaction.response.send_message(embed=embed, ephemeral=True)

    except Exception as e:
        await interaction.response.send_message(f"Fehler beim Loggen des Trades: {e}", ephemeral=True)
        print(f"Fehler beim Loggen des Trades für {interaction.user.name}: {e}")

@bot.tree.command(name="trades", description="Zeigt deine letzten geloggten Trades an.")
@app_commands.describe(
    limit="Anzahl der Trades, die angezeigt werden sollen (Standard: 5)"
)
async def view_trades(interaction: discord.Interaction, limit: int = 5):
    """Slash-Befehl zum Anzeigen der geloggten Trades."""
    try:
        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            "SELECT id, asset, direction, entry_price, exit_price, pnl_amount, pnl_percent, comment, image_filename, timestamp FROM trades WHERE user_id = ? ORDER BY timestamp DESC LIMIT ?",
            (interaction.user.id, limit)
        )
        trades = cursor.fetchall()
        conn.close()

        if not trades:
            await interaction.response.send_message("Du hast noch keine Trades geloggt.", ephemeral=True)
            return

        embed = discord.Embed(
            title=f"📊 Deine letzten {len(trades)} Trades",
            color=discord.Color.blue()
        )
        for trade in trades:
            image_info = "📷 Bild vorhanden" if trade['image_filename'] else "Kein Bild"

            embed.add_field(
                name=f"{trade['asset']} ({trade['direction']})",
                value=(
                    f"Einstieg: ${trade['entry_price']:,.2f} | Ausstieg: ${trade['exit_price']:,.2f}\n"
                    f"P&L: ${trade['pnl_amount']:,.2f} ({trade['pnl_percent']:.2f}%)\n"
                    f"Zeitpunkt: {datetime.datetime.fromisoformat(trade['timestamp']).strftime('%Y-%m-%d %H:%M')}\n"
                    f"Kommentar: {trade['comment'] if trade['comment'] else 'Kein Kommentar'}\n"
                    f"*{image_info}*"
                ),
                inline=False
            )
        await interaction.response.send_message(embed=embed, ephemeral=True)

    except Exception as e:
        await interaction.response.send_message(f"Fehler beim Abrufen der Trades: {e}", ephemeral=True)
        print(f"Fehler beim Abrufen der Trades für {interaction.user.name}: {e}")

# --- NEU: Postmortem System ---
class PostmortemView(discord.ui.View):
    """Eine View-Klasse für die Interaktion mit Postmortems (Buttons zum Abstimmen)."""
    def __init__(self, postmortem_id, author_id):
        super().__init__(timeout=None)
        self.postmortem_id = postmortem_id
        self.author_id = author_id

        # Dynamische Button-Erstellung und Callback-Zuweisung
        btn_good = discord.ui.Button(label="Guter Trade", style=discord.ButtonStyle.green, custom_id=f"postmortem_vote_good_{postmortem_id}")
        btn_good.callback = self._vote_callback # _vote_callback wird alle Abstimmungen verarbeiten
        self.add_item(btn_good)

        btn_needs_improvement = discord.ui.Button(label="Verbesserungswürdig", style=discord.ButtonStyle.blurple, custom_id=f"postmortem_vote_needs_improvement_{postmortem_id}")
        btn_needs_improvement.callback = self._vote_callback
        self.add_item(btn_needs_improvement)

        btn_bad = discord.ui.Button(label="Schlechter Trade", style=discord.ButtonStyle.red, custom_id=f"postmortem_vote_bad_{postmortem_id}")
        btn_bad.callback = self._vote_callback
        self.add_item(btn_bad)

        btn_show_results = discord.ui.Button(label="Ergebnisse zeigen & schließen", style=discord.ButtonStyle.grey, custom_id="postmortem_show_results_button")
        btn_show_results.callback = self._show_results_callback # Separater Callback für diesen Button
        self.add_item(btn_show_results)

    async def _vote_callback(self, interaction: discord.Interaction):
        """Generischer Callback für die Abstimmungs-Buttons."""
        await interaction.response.defer(ephemeral=True)

        parts = interaction.data['custom_id'].split('_')
        vote_type = parts[2] # 'good', 'needs_improvement', 'bad'
        postmortem_msg_id = int(parts[3])
        user_id = interaction.user.id

        if postmortem_msg_id not in active_postmortems:
            await interaction.followup.send("Dieses Postmortem ist nicht mehr aktiv.", ephemeral=True)
            return

        postmortem_data = active_postmortems[postmortem_msg_id]

        if user_id in postmortem_data['votes']:
            old_vote = postmortem_data['votes'][user_id]
            if old_vote == vote_type:
                await interaction.followup.send(f"Du hast bereits mit '{vote_type.replace('_', ' ').title()}' abgestimmt.", ephemeral=True)
                return
            else:
                postmortem_data['votes'][user_id] = vote_type
                await interaction.followup.send(f"Deine Stimme wurde von '{old_vote.replace('_', ' ').title()}' zu '{vote_type.replace('_', ' ').title()}' geändert.", ephemeral=True)
        else:
            postmortem_data['votes'][user_id] = vote_type
            await interaction.followup.send(f"Du hast mit '{vote_type.replace('_', ' ').title()}' abgestimmt. Danke für dein Feedback!", ephemeral=True)

            if is_mentor(interaction.user):
                await add_xp(interaction.user.id, 10) # XP für Mentoren, die abstimmen

    async def _show_results_callback(self, interaction: discord.Interaction): # Callback-Name für den "Ergebnisse zeigen" Button
        """Callback für den "Ergebnisse zeigen & schließen" Button."""
        await interaction.response.defer()

        if self.postmortem_id not in active_postmortems:
            await interaction.followup.send("Dieses Postmortem ist nicht mehr aktiv.", ephemeral=True)
            return

        is_author = interaction.user.id == self.author_id
        is_admin_or_mod = is_moderator(interaction.user)

        if not (is_author or is_admin_or_mod):
            await interaction.followup.send("Nur der Ersteller oder ein Admin/Mod kann die Ergebnisse anzeigen.", ephemeral=True)
            return

        await self.end_postmortem_and_show_results(interaction.message, interaction)

    async def end_postmortem_and_show_results(self, message, interaction=None):
        """Beendet das Postmortem, zeigt Ergebnisse an und deaktiviert Buttons."""
        if message.id not in active_postmortems:
            if interaction:
                await interaction.followup.send("Dieses Postmortem ist nicht mehr aktiv.", ephemeral=True)
            return

        postmortem_data = active_postmortems.pop(message.id)
        votes = postmortem_data['votes']

        results = {
            'good': 0,
            'needs_improvement': 0,
            'bad': 0
        }
        for vote_type in votes.values():
            if vote_type in results:
                results[vote_type] += 1

        total_votes = sum(results.values())

        results_text = (
            f"**Guter Trade**: {results['good']} Stimmen\n"
            f"**Verbesserungswürdig**: {results['needs_improvement']} Stimmen\n"
            f"**Schlechter Trade**: {results['bad']} Stimmen\n"
            f"**Gesamtstimmen**: {total_votes}"
        )

        original_embed = message.embeds[0]
        original_embed.title = f"📈 Postmortem Abgeschlossen: {original_embed.title.replace('Postmortem:', '').strip()}"
        original_embed.description = original_embed.description + "\n\n**Abstimmungsergebnisse:**\n" + results_text
        original_embed.color = discord.Color.dark_grey()
        original_embed.set_footer(text=f"Postmortem geschlossen | Ursprünglich gepostet von {original_embed.footer.text.split('von ')[1]}")

        for item in self.children:
            item.disabled = True

        await message.edit(embed=original_embed, view=self)
        if interaction:
            try:
                await interaction.followup.send("Postmortem beendet und Ergebnisse angezeigt!")
            except discord.HTTPException:
                pass


@bot.tree.command(name="postmortem", description="Poste einen Trade zur Community-Diskussion und Abstimmung.")
@app_commands.describe(
    asset="Das gehandelte Asset (z.B. BTC, ETH)",
    direction="Die Richtung des Trades (Long/Short)",
    pnl_amount="Dein Gewinn/Verlust in Währung",
    pnl_percent="Dein Gewinn/Verlust in Prozent",
    reasoning="Deine Begründung für den Trade, deine Analyse, was gelernt wurde.",
    image_file="Ein optionales Bild deines Trades (Chart, etc.)"
)
@app_commands.choices(
    direction=[
        app_commands.Choice(name="Long", value="Long"),
        app_commands.Choice(name="Short", value="Short"),
    ]
)
async def postmortem_command(
    interaction: discord.Interaction,
    asset: str,
    direction: str,
    pnl_amount: float,
    pnl_percent: float,
    reasoning: str,
    image_file: discord.Attachment = None
):
    """Slash-Befehl zum Posten eines Postmortems."""
    if POSTMORTEM_CHANNEL_ID == 0:
        await interaction.response.send_message("Der Postmortem-Kanal ist noch nicht konfiguriert. Bitte Admin kontaktieren.", ephemeral=True)
        return

    postmortem_channel = bot.get_channel(POSTMORTEM_CHANNEL_ID)
    if not postmortem_channel:
        await interaction.response.send_message(f"Der konfigurierte Postmortem-Kanal mit ID {POSTMORTEM_CHANNEL_ID} wurde nicht gefunden.", ephemeral=True)
        print(f"⚠️ Postmortem-Kanal mit ID {POSTMORTEM_CHANNEL_ID} nicht gefunden.")
        return

    await interaction.response.defer(ephemeral=True)

    try:
        image_filename = None
        file_to_send = None
        if image_file:
            timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
            safe_asset = ''.join(char for char in asset if char.isalnum())
            image_filename = f"postmortem_{interaction.user.id}_{safe_asset}_{timestamp}.png"
            image_path = os.path.join(TRADE_IMAGE_DIR, image_filename)
            await image_file.save(image_path)
            print(f"Postmortem Bild gespeichert: {image_path}")
            file_to_send = discord.File(image_path, filename=image_filename)

        pnl_color = discord.Color.green() if pnl_amount >= 0 else discord.Color.red()

        postmortem_embed = discord.Embed(
            title=f"📈 Postmortem: {asset} ({direction})",
            description=f"**Gewinn/Verlust:** ${pnl_amount:,.2f} ({pnl_percent:.2f}%)\n\n"
                        f"**Begründung/Analyse:**\n{reasoning}\n\n"
                        f"**Stimme ab, wie du diesen Trade findest!**",
            color=pnl_color
        )
        postmortem_embed.set_author(name=f"Postmortem von {interaction.user.display_name}", icon_url=interaction.user.display_avatar.url)
        postmortem_embed.set_footer(text=f"ID: {interaction.user.id} | {datetime.datetime.now().strftime('%Y-%m-%d %H:%M')}")

        if file_to_send:
            postmortem_embed.set_image(url=f"attachment://{image_filename}")

        view = PostmortemView(0, interaction.user.id) # postmortem_id wird nach dem Senden der Nachricht aktualisiert

        sent_message = await postmortem_channel.send(embed=postmortem_embed, view=view, file=file_to_send)

        conn = get_db_connection()
        cursor = conn.cursor()
        cursor.execute(
            """
            INSERT INTO postmortems (user_id, message_id, asset, direction, pnl_amount, pnl_percent, reasoning, image_filename, timestamp)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            """,
            (
                interaction.user.id,
                sent_message.id,
                asset,
                direction,
                pnl_amount,
                pnl_percent,
                reasoning,
                image_filename,
                datetime.datetime.now().isoformat()
            )
        )
        conn.commit()
        conn.close()

        active_postmortems[sent_message.id] = {
            'author_id': interaction.user.id,
            'votes': {}
        }
        view.postmortem_id = sent_message.id # Aktualisiere die postmortem_id in der View-Instanz

        await interaction.followup.send(f"Dein Postmortem wurde erfolgreich im Kanal {postmortem_channel.mention} gepostet!", ephemeral=True)

    except Exception as e:
        await interaction.followup.send(f"Fehler beim Posten des Postmortems: {e}", ephemeral=True)
        print(f"Fehler beim Posten des Postmortems für {interaction.user.name}: {e}")

# --- Krypto-Nachrichten Funktionen ---
async def get_crypto_news():
    """Ruft aktuelle Krypto-Nachrichten von NewsAPI ab."""
    if not NEWSAPI_KEY:
        print("⚠️ NEWSAPI_KEY nicht in den Umgebungsvariablen gefunden. /news Befehl wird nicht funktionieren.")
        return None

    base_url = "https://newsapi.org/v2/everything"
    params = {
        'q': 'Bitcoin OR Ethereum OR Crypto',
        'sortBy': 'publishedAt',
        'apiKey': NEWSAPI_KEY,
        'language': 'de'
    }
    async with aiohttp.ClientSession() as session:
        try:
            async with session.get(base_url, params=params) as response:
                if response.status == 200:
                    data = await response.json()
                    return data.get('articles', [])[:5]
                else:
                    print(f"Fehler beim Abrufen der Nachrichten von NewsAPI: {response.status} - {await response.text()}")
                    return None
        except aiohttp.ClientError as e:
            print(f"AIOHTTP Fehler beim Abrufen der Nachrichten: {e}")
            return None
        except Exception as e:
            print(f"Unerwarteter Fehler beim Abrufen der Nachrichten: {e}")
            return None

@bot.tree.command(name="news", description="Zeigt die neuesten Krypto-Nachrichten.")
async def crypto_news(interaction: discord.Interaction):
    """Slash-Befehl zum Anzeigen aktueller Krypto-Nachrichten."""
    await interaction.response.defer()
    news_articles = await get_crypto_news()
    if news_articles:
        embed = discord.Embed(
            title="📰 Recent News in Crypto🔥",
            color=discord.Color.orange()
        )
        for article in news_articles:
            title = article.get('title', 'Kein Titel')
            url = article.get('url')
            source_name = article.get('source', {}).get('name', 'Unbekannt')
            published_at_str = article.get('publishedAt')

            title = title.replace('&#039;', "'").replace('&quot;', '"').replace('&amp;', '&')

            time_ago_str = ""
            if published_at_str:
                try:
                    published_time = datetime.datetime.fromisoformat(published_at_str.replace('Z', '+00:00'))
                    now = datetime.datetime.now(datetime.timezone.utc)
                    time_diff = now - published_time

                    if time_diff.total_seconds() < 60:
                        time_ago_str = f"{int(time_diff.total_seconds())} seconds ago"
                    elif time_diff.total_seconds() < 3600:
                        time_ago_str = f"{int(time_diff.total_seconds() / 60)} minutes ago"
                    elif time_diff.total_seconds() < 86400:
                        time_ago_str = f"{int(time_diff.total_seconds() / 3600)} hours ago"
                    else:
                        time_ago_str = f"{int(time_diff.total_seconds() / 86400)} days ago"

                except ValueError:
                    time_ago_str = "Unknown time"

            field_value = f"{time_ago_str} via {source_name} [➡️]({url})"
            embed.add_field(name=title, value=field_value, inline=False)

        class NewsButtons(discord.ui.View):
            def __init__(self):
                super().__init__(timeout=None)
                # Die Buttons "Popular", "Positive", "Negative" sind hier nur Platzhalter,
                # da ihre Funktionalität eine komplexere NewsAPI-Integration erfordern würde
                self.add_item(discord.ui.Button(label="Popular", style=discord.ButtonStyle.secondary, emoji="🔥", custom_id="news_popular", disabled=True))
                # Der "Recent" Button ist hier nicht dupliziert, da er oben schon als Callback definiert ist
                self.add_item(discord.ui.Button(label="Positive", style=discord.ButtonStyle.secondary, emoji="❤️", custom_id="news_positive", disabled=True))
                self.add_item(discord.ui.Button(label="negative", style=discord.ButtonStyle.secondary, emoji="😠", custom_id="news_negative", disabled=True))

                # Dynamisch erstellter "Recent" Button
                btn_recent = discord.ui.Button(label="Recent", style=discord.ButtonStyle.secondary, emoji="🕐", custom_id="news_recent_refresh")
                btn_recent.callback = self.recent_button_callback
                self.add_item(btn_recent)

            # Callback für den "Recent" Button (ohne @discord.ui.button Dekorator)
            async def recent_button_callback(self, interaction: discord.Interaction):
                await interaction.response.defer()
                updated_news_articles = await get_crypto_news()
                if updated_news_articles:
                    updated_embed = discord.Embed(
                        title="📰 Recent News in Crypto🔥 (Aktualisiert)",
                        color=discord.Color.orange()
                    )
                    for article in updated_news_articles:
                        title = article.get('title', 'Kein Titel')
                        url = article.get('url')
                        source_name = article.get('source', {}).get('name', 'Unbekannt')
                        published_at_str = article.get('publishedAt')

                        title = title.replace('&#039;', "'").replace('&quot;', '"').replace('&amp;', '&')

                        time_ago_str = ""
                        if published_at_str:
                            try:
                                published_time = datetime.datetime.fromisoformat(published_at_str.replace('Z', '+00:00'))
                                now = datetime.datetime.now(datetime.timezone.utc)
                                time_diff = now - published_time

                                if time_diff.total_seconds() < 60:
                                    time_ago_str = f"{int(time_diff.total_seconds())} seconds ago"
                                elif time_diff.total_seconds() < 3600:
                                    time_ago_str = f"{int(time_diff.total_seconds() / 60)} minutes ago"
                                elif time_diff.total_seconds() < 86400:
                                    time_ago_str = f"{int(time_diff.total_seconds() / 3600)} hours ago"
                                else:
                                    time_ago_str = f"{int(time_diff.total_seconds() / 86400)} days ago"

                            except ValueError:
                                time_ago_str = "Unknown time"

                        field_value = f"{time_ago_str} via {source_name} [➡️]({url})"
                        updated_embed.add_field(name=title, value=field_value, inline=False)
                    await interaction.edit_original_response(embed=updated_embed, view=self)
                    await interaction.followup.send("Nachrichten aktualisiert!", ephemeral=True)
                else:
                    await interaction.followup.send("Konnte Nachrichten nicht aktualisieren.", ephemeral=True)


        await interaction.followup.send(embed=embed, view=NewsButtons())
    else:
        await interaction.followup.send("Es konnten keine aktuellen Krypto-Nachrichten abgerufen werden. Bitte stelle sicher, dass der `NEWSAPI_KEY` korrekt gesetzt ist und funktioniert.", ephemeral=True)

# --- NEU: Mentoren-XP-System Commands ---

@bot.tree.command(name="mentor_idee", description="Teile einen Trading-Tipp oder eine Erkenntnis als Mentor und sammle XP.")
@app_commands.describe(
    tip="Dein Trading-Tipp oder deine Erkenntnis.",
    category="Die Kategorie deines Tipps (z.B. Strategie, Psychologie, Risiko)."
)
@app_commands.choices(
    category=[
        app_commands.Choice(name="Strategie", value="Strategie"),
        app_commands.Choice(name="Psychologie", value="Psychologie"),
        app_commands.Choice(name="Risikomanagement", value="Risikomanagement"),
        app_commands.Choice(name="Marktanalyse", value="Marktanalyse"),
        app_commands.Choice(name="Allgemein", value="Allgemein"),
    ]
)
async def mentor_idee_command(
    interaction: discord.Interaction,
    tip: str,
    category: str
):
    """Slash-Befehl für Mentoren, um Tipps zu teilen und XP zu erhalten."""
    if not is_mentor(interaction.user):
        await interaction.response.send_message("Dieser Befehl ist nur für Mentoren verfügbar.", ephemeral=True)
        return

    xp_gained = 25
    await add_xp(interaction.user.id, xp_gained)

    embed = discord.Embed(
        title=f"💡 Mentoren-Tipp: {category}",
        description=tip,
        color=discord.Color.gold()
    )
    embed.set_author(name=f"Von Mentor {interaction.user.display_name}", icon_url=interaction.user.display_avatar.url)
    embed.set_footer(text=f"XP erhalten: {xp_gained} | {datetime.datetime.now().strftime('%Y-%m-%d %H:%M')}")

    await interaction.response.send_message(
        f"Dein Tipp wurde geteilt und du hast **{xp_gained} XP** erhalten! 🎉",
        embed=embed
    )


@bot.tree.command(name="myxp", description="Zeigt deine aktuellen Mentoren-XP an (nur für Mentoren).")
async def myxp_command(interaction: discord.Interaction):
    """Zeigt die XP des Mentors an."""
    if not is_mentor(interaction.user):
        await interaction.response.send_message("Dieser Befehl ist nur für Mentoren verfügbar.", ephemeral=True)
        return

    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT xp FROM mentor_xp WHERE user_id = ?", (interaction.user.id,))
    result = cursor.fetchone()
    conn.close()

    if result:
        xp = result['xp']
        embed = discord.Embed(
            title="✨ Deine Mentoren-XP",
            description=f"Du hast aktuell **{xp} XP** als Mentor!",
            color=discord.Color.gold()
        )
        embed.set_footer(text="Weiter so! Dein Wissen ist Gold wert.")
        await interaction.response.send_message(embed=embed, ephemeral=True)
    else:
        await interaction.response.send_message("Du hast noch keine Mentoren-XP gesammelt.", ephemeral=True)


@bot.tree.command(name="mentorleaderboard", description="Zeigt die Top-Mentoren nach XP an (nur für Mentoren).")
@app_commands.describe(
    limit="Anzahl der Mentoren, die angezeigt werden sollen (Standard: 10)"
)
async def mentor_leaderboard_command(interaction: discord.Interaction, limit: int = 10):
    """Zeigt die Top-Mentoren nach XP an."""
    if not is_mentor(interaction.user):
        await interaction.response.send_message("Dieser Befehl ist nur für Mentoren verfügbar.", ephemeral=True)
        return

    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT user_id, xp FROM mentor_xp ORDER BY xp DESC LIMIT ?", (limit,))
    results = cursor.fetchall()
    conn.close()

    if not results:
        await interaction.response.send_message("Es wurden noch keine Mentoren-XP gesammelt.", ephemeral=True)
        return

    leaderboard_text = ""
    for i, row in enumerate(results):
        user_id = row['user_id']
        user = bot.get_user(user_id)
        username = user.display_name if user else f"Unbekannter Benutzer ({user_id})"
        leaderboard_text += f"{i+1}. **{username}**: {row['xp']} XP\n"

    embed = discord.Embed(
        title="🏆 Top Mentoren-Leaderboard",
        description=leaderboard_text,
        color=discord.Color.blue()
    )
    embed.set_footer(text="Die aktivsten Wissensgeber unserer Community!")
    await interaction.response.send_message(embed=embed)


@bot.tree.command(name="addxp_manual", description="Vergibt manuell XP an einen Mentor (nur für Administratoren).")
@app_commands.describe(
    mentor_id="Die Discord User ID des Mentors",
    xp_amount="Die Menge an XP, die vergeben werden soll",
    reason="Der Grund für die XP-Vergabe"
)
async def addxp_manual_command(
    interaction: discord.Interaction,
    mentor_id: str,
    xp_amount: int,
    reason: str
):
    """Slash-Befehl zur manuellen XP-Vergabe an Mentoren."""
    if not interaction.user.guild_permissions.administrator:
        await interaction.response.send_message("Diesen Befehl können nur Administratoren verwenden.", ephemeral=True)
        return

    try:
        target_id = int(mentor_id)
    except ValueError:
        await interaction.response.send_message("Ungültige Mentor ID. Bitte gib eine gültige numerische Discord User ID ein.", ephemeral=True)
        return

    target_member = interaction.guild.get_member(target_id)
    if not target_member:
        await interaction.response.send_message(f"Benutzer mit ID {mentor_id} auf diesem Server nicht gefunden.", ephemeral=True)
        return
    if not is_mentor(target_member):
        await interaction.response.send_message(f"Benutzer {target_member.display_name} ist kein Mentor. XP werden trotzdem vergeben, aber es ist unüblich.", ephemeral=True)

    await add_xp(target_id, xp_amount)

    target_user = bot.get_user(target_id)
    target_username = target_user.display_name if target_user else f"Benutzer mit ID {target_id}"

    embed = discord.Embed(
        title="➕ Manuelle XP-Vergabe",
        description=f"**{xp_amount} XP** wurden an **{target_username}** vergeben.",
        color=discord.Color.blue()
    )
    embed.add_field(name="Grund", value=reason, inline=False)
    embed.set_footer(text=f"Vergeben von {interaction.user.display_name} | {datetime.datetime.now().strftime('%Y-%m-%d %H:%M')}")

    await interaction.response.send_message(embed=embed, ephemeral=True)


# Startet den Bot
if __name__ == "__main__":
    if TOKEN:
        bot.run(TOKEN)
    else:
        print("Fehler: DISCORD_BOT_TOKEN nicht gefunden. Bitte stelle sicher, dass es in deiner .env-Datei korrekt gesetzt ist.")
