import discord
from discord.ext import commands
from discord import app_commands
import os
import datetime
import traceback
from flask import Flask
from threading import Thread

app = Flask(__name__)

@app.route('/')
def home():
    return "Bot is alive!"

def run():
    app.run(host='0.0.0.0', port=8080)

def keep_alive():
    t = Thread(target=run)
    t.start()

# Constants
ABSENCE_CHANNEL_ID = 1361195118731722841  # Replace with your actual channel ID
ALLOWED_ROLE_IDS = [
    1333821999155380345,  # Replace with your allowed role IDs
    1347052625736110231,
    1333821257619079168,
    1341709228624318474
]

intents = discord.Intents.default()
intents.messages = True
intents.guilds = True
intents.members = True
intents.message_content = True

class AbsenceBot(commands.Bot):
    def __init__(self):
        super().__init__(command_prefix=None, intents=intents)
        self.absences = {}  # user_id: {end_time, reason}

    async def setup_hook(self):
        await self.add_cog(Absence(self))
        await self.tree.sync()  # Sync slash commands
        print("Slash commands synced.")

bot = AbsenceBot()

class Absence(commands.Cog):
    def __init__(self, bot: AbsenceBot):
        self.bot = bot

    @app_commands.command(name="absence", description="Notify the server you're going to be absent.")
    @app_commands.describe(
        time="How long will you be absent? (e.g., 2d 5h 30m)",
        reason="Why are you going to be absent?"
    )
    async def absence(self, interaction: discord.Interaction, time: str, reason: str):
        if not any(role.id in ALLOWED_ROLE_IDS for role in interaction.user.roles):
            await interaction.response.send_message("You don't have permission to use this command.", ephemeral=True)
            return

        duration = parse_time_string(time)
        if duration is None:
            await interaction.response.send_message("Invalid time format. Use things like `2d 5h 30m`.", ephemeral=True)
            return

        end_time = datetime.datetime.utcnow() + duration
        bot.absences[interaction.user.id] = {"end_time": end_time, "reason": reason}

        channel = bot.get_channel(ABSENCE_CHANNEL_ID)
        if channel:
            await channel.send(
                f"**📢 ABSENCE NOTICE**\n\n"
                f"**User:** {interaction.user.mention}\n"
                f"**Time Away:** {time}\n"
                f"**Reason:** {reason}"
            )
            await interaction.response.send_message("Your absence has been posted.", ephemeral=True)
        else:
            await interaction.response.send_message("Could not find the absence channel.", ephemeral=True)

    @commands.Cog.listener()
    async def on_message(self, message):
        if message.author.bot:
            return

        # Auto-remove absence if user sends a message
        if message.author.id in bot.absences:
            del bot.absences[message.author.id]
            await message.channel.send(f"Welcome back {message.author.mention}! I’ve removed your **absence**.")

        # Ping detection
        for user_id, absence in bot.absences.items():
            if f"<@{user_id}>" in message.content or f"<@!{user_id}>" in message.content:
                reason = absence['reason']
                remaining = absence['end_time'] - datetime.datetime.utcnow()
                time_left = format_timedelta(remaining)
                user = message.guild.get_member(user_id)
                if user:
                    await message.channel.send(
                        f"{user.mention} is **absent** for \"{reason}\", they will come back in {time_left}."
                    )

# Utils
def parse_time_string(time_str):
    try:
        days = hours = minutes = 0
        for part in time_str.split():
            if "d" in part:
                days += int(part.replace("d", ""))
            elif "h" in part:
                hours += int(part.replace("h", ""))
            elif "m" in part:
                minutes += int(part.replace("m", ""))
        return datetime.timedelta(days=days, hours=hours, minutes=minutes)
    except:
        return None

def format_timedelta(delta):
    seconds = int(delta.total_seconds())
    if seconds < 0:
        return "less than a minute"
    days, seconds = divmod(seconds, 86400)
    hours, seconds = divmod(seconds, 3600)
    minutes = seconds // 60
    parts = []
    if days > 0:
        parts.append(f"{days}d")
    if hours > 0:
        parts.append(f"{hours}h")
    if minutes > 0:
        parts.append(f"{minutes}m")
    return " ".join(parts) if parts else "less than a minute"

# Start the bot with error handling
try:
    token = os.getenv("DISCORD_TOKEN")
    if not token:
        raise ValueError("DISCORD_TOKEN not found in environment variables! Add it to Secrets.")
        
    @bot.event
    async def on_ready():
        print(f"Logged in as {bot.user}")
        print("Bot is ready and commands are synced.")
        
    print("Starting the keep-alive server...")
    keep_alive()
    print("Starting the bot...")
    bot.run(token)
except Exception as e:
    print(f"Error while running the bot: {e}")
    traceback.print_exc()
