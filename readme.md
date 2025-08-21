# 🧙‍♂️ AzerothCore Discord Status Bot

This guide walks you through creating a Discord bot that updates a channel's name with your AzerothCore server's online status and player count.

---

## 1. Create a Discord Application & Bot

1. **Navigate to the [Discord Developer Portal](https://discord.com/developers/applications).**
2. Click **"New Application"** and name it (e.g., `AzerothCore Status Bot`).
3. In the left sidebar, go to **"Bot"** → **"Add Bot"**.
4. Under **"Privileged Gateway Intents"**, enable:

   * `Presence Intent`
   * `Server Members Intent`
   * `Message Content Intent`
5. **Copy the Bot Token** — you'll need this for your `.env` file.
6. Under **"OAuth2"** → **"URL Generator"**:

   * Select **"bot"** and **"applications.commands"** in the **Scopes**.
   * Under **"Bot Permissions"**, select:

     * `Manage Channels`
     * `Read Messages`
     * `Send Messages`
     * `Manage Messages`
   * Copy the generated URL and paste it into your browser to invite the bot to your server.

---

## 2. Prepare Your Server Environment

Ensure your server has:

* **Python 3.9+** installed.
* **pip** package manager.
* Access to your **AzerothCore database** (`acore_characters`).
* **MySQL or MariaDB** client installed.

---

## 3. Set Up Your Project Directory

```bash
mkdir acore-discord-bot
cd acore-discord-bot
```

(Optional) Create a virtual environment:

```bash
python -m venv venv
# Activate the virtual environment
# Linux/macOS
source venv/bin/activate
# Windows
venv\Scripts\activate
```

---

## 4. Install Required Python Packages

```bash
pip install discord.py aiomysql python-dotenv
```

---

## 5. Configure Environment Variables

Create a `.env` file in your project directory with the following content:

```env
DISCORD_TOKEN=YOUR_BOT_TOKEN
CHANNEL_ID=YOUR_DISCORD_CHANNEL_ID
DB_HOST=127.0.0.1
DB_USER=acore_user
DB_PASS=your_db_password
DB_NAME=acore_characters
REALM_IP=127.0.0.1
REALM_PORT=8085
UPDATE_INTERVAL=60
```

Replace the placeholders with your actual values.

---

## 6. Create the Bot Script

Create a file named `bot.py` in your project directory and add the following code:

```python
import discord
import asyncio
import aiomysql
import os
from dotenv import load_dotenv
import logging

# -------------------- CONFIG --------------------
load_dotenv()

TOKEN = os.getenv("DISCORD_TOKEN")
CHANNEL_ID = int(os.getenv("CHANNEL_ID"))
DB_CONFIG = {
    "host": os.getenv("DB_HOST", "127.0.0.1"),
    "user": os.getenv("DB_USER", "acore_user"),
    "password": os.getenv("DB_PASS", "password"),
    "db": os.getenv("DB_NAME", "acore_characters")
}
REALM_IP = os.getenv("REALM_IP", "127.0.0.1")
REALM_PORT = int(os.getenv("REALM_PORT", 8085))
UPDATE_INTERVAL = int(os.getenv("UPDATE_INTERVAL", 60))
# ------------------------------------------------

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[logging.FileHandler("status_bot.log"), logging.StreamHandler()]
)

intents = discord.Intents.default()
client = discord.Client(intents=intents)

async def check_realm_online():
    try:
        fut = asyncio.open_connection(REALM_IP, REALM_PORT)
        reader, writer = await asyncio.wait_for(fut, timeout=3)
        writer.close()
        await writer.wait_closed()
        return True
    except:
        return False

async def get_players_online():
    try:
        pool = await aiomysql.create_pool(**DB_CONFIG)
        async with pool.acquire() as conn:
            async with conn.cursor() as cur:
                await cur.execute("SELECT COUNT(*) FROM characters WHERE online=1;")
                count = (await cur.fetchone())[0]
        pool.close()
        await pool.wait_closed()
        logging.info(f"Players online: {count}")
        return count
    except Exception as e:
        logging.error(f"DB query failed: {e}")
        return 0

async def update_status():
    await client.wait_until_ready()
    channel = client.get_channel(CHANNEL_ID)
    if not channel:
        logging.error("Channel not found. Check CHANNEL_ID.")
        return

    while not client.is_closed():
        try:
            online = await check_realm_online()
            if online:
                players = await get_players_online()
                player_text = "Player" if players == 1 else "Players"
                new_name = f"Online — {players} {player_text}"
            else:
                new_name = "Offline"

            if channel.name != new_name:
                await channel.edit(name=new_name)
                logging.info(f"Updated channel: {new_name}")

        except Exception as e:
            logging.error(f"Error updating channel: {e}")

        await asyncio.sleep(UPDATE_INTERVAL)

@client.event
async def on_ready():
    logging.info(f"Bot logged in as {client.user}")
    client.loop.create_task(update_status())

client.run(TOKEN)
```

---

## 7. Run the Bot

```bash
python bot.py
```

The bot will log in, and the specified Discord channel's name will update based on your AzerothCore server's status.

---

## 8. Optional: Run the Bot as a Background Service

**On Linux/macOS using PM2:**

```bash
npm install pm2 -g
pm2 start bot.py --interpreter python3 --name acore-status-bot
pm2 startup
pm2 save
```

**On Windows**, use Task Scheduler to run `python bot.py` at startup.

---

## 9. Troubleshooting

* **Channel not updating:** Ensure the bot has the `Manage Channels` permission.
* **DB connection failed:** Verify `.env` credentials and database accessibility.
* **Realm offline:** Confirm `REALM_IP` and `REALM_PORT` are correct.

---

## 10. Notes

* The bot edits the **channel name** to show server status.
* Player counts are fetched **directly from the `characters` table**.
* Logs are saved to `status_bot.log` for debugging.

[Creating a Discord Bot in Python (2025)](https://www.youtube.com/watch?v=YD_N6Ffoojw&utm_source=chatgpt.com)

---
