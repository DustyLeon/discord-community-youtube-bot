import discord
from discord.ext import tasks, commands
from discord import app_commands
import sqlite3
import feedparser
import aiohttp
import random
from typing import Optional
from googleapiclient.discovery import build

# --- CONFIGURATION ---
# IMPORTANT: Replace these placeholders with your actual keys. Never share them publicly!
DISCORD_TOKEN = "YOUR_DISCORD_BOT_TOKEN_HERE"
YOUTUBE_API_KEY = "YOUR_YOUTUBE_API_KEY_HERE"
CHECK_INTERVAL_MINUTES = 15 

# Initialize YouTube API client
youtube = build('youtube', 'v3', developerKey=YOUTUBE_API_KEY)

# Initialize Discord Bot
class MyBot(commands.Bot):
    def __init__(self):
        intents = discord.Intents.default()
        intents.message_content = True
        super().__init__(command_prefix="!", intents=intents)

    async def setup_hook(self):
        await self.tree.sync()
        check_new_videos.start()

bot = MyBot()

# --- DATABASE SETUP ---
def setup_db():
    conn = sqlite3.connect('channels.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS channels 
                 (channel_id TEXT PRIMARY KEY, channel_name TEXT, 
                  last_video_id TEXT, discord_channel_id INTEGER)''')
    conn.commit()
    conn.close()

setup_db()

# --- HELPER FUNCTIONS ---
async def is_short(video_id):
    """
    Hack to detect YouTube Shorts. The API doesn't tell us, but requesting 
    /shorts/VIDEO_ID returns a 200 OK if it's a short, and redirects (303) to /watch if it's a normal video.
    """
    url = f"https://www.youtube.com/shorts/{video_id}"
    async with aiohttp.ClientSession() as session:
        async with session.head(url, allow_redirects=False) as response:
            return response.status == 200

def get_channel_info(channel_query):
    """Searches YouTube API for a channel by Name, Handle, URL, or ID."""
    channel_query = channel_query.strip()
    
    # 1. Parse URLs to extract Handle or Channel ID
    if "youtube.com" in channel_query or "youtu.be" in channel_query:
        if "/channel/" in channel_query:
            channel_query = channel_query.split("/channel/")[1].split("/")[0].split("?")[0]
        elif "/@" in channel_query:
            channel_query = "@" + channel_query.split("/@")[1].split("/")[0].split("?")[0]
        elif "/c/" in channel_query or "/user/" in channel_query:
            # Legacy URL fallback
            channel_query = channel_query.split("/")[-1].split("?")[0]
            
    # 2. Check if it's an exact Channel ID (starts with UC, 24 characters long)
    if channel_query.startswith("UC") and len(channel_query) == 24:
        request = youtube.channels().list(part="snippet", id=channel_query)
        response = request.execute()
        if response.get("items"):
            item = response["items"][0]
            return item["id"], item["snippet"]["title"]
            
    # 3. Check if it's a Handle (with or without @)
    # Automatically add the @ if the user forgot it to test exact handle matches first
    handle_to_test = channel_query if channel_query.startswith("@") else f"@{channel_query}"
    try:
        request = youtube.channels().list(part="snippet", forHandle=handle_to_test)
        response = request.execute()
        if response.get("items"):
            item = response["items"][0]
            return item["id"], item["snippet"]["title"]
    except Exception as e:
        print(f"Handle search failed for {handle_to_test}: {e}")
        pass # If forHandle endpoint fails, fallback to general search
            
    # 4. Fallback to general search
    request = youtube.search().list(part="snippet", type="channel", q=channel_query, maxResults=1)
    response = request.execute()
    
    if not response.get("items"):
        return None, None
    
    item = response["items"][0]
    return item["snippet"]["channelId"], item["snippet"]["title"]

# --- BACKGROUND TASK ---
@tasks.loop(minutes=CHECK_INTERVAL_MINUTES)
async def check_new_videos():
    """Polls RSS feeds for new videos every X minutes."""
    conn = sqlite3.connect('channels.db')
    c = conn.cursor()
    c.execute("SELECT channel_id, channel_name, last_video_id, discord_channel_id FROM channels")
    rows = c.fetchall()
    
    for row in rows:
        channel_id, channel_name, last_video_id, discord_channel_id = row
        feed_url = f"https://www.youtube.com/feeds/videos.xml?channel_id={channel_id}"
        feed = feedparser.parse(feed_url)
        
        if not feed.entries:
            continue
            
        latest_video = feed.entries[0]
        latest_video_id = latest_video.yt_videoid
        
        # If the video is new, process it
        if latest_video_id != last_video_id:
            # Update DB immediately so we don't get stuck in a loop if something fails
            c.execute("UPDATE channels SET last_video_id = ? WHERE channel_id = ?", (latest_video_id, channel_id))
            conn.commit()
            
            # Check if it's a short. If not, post it!
            if not await is_short(latest_video_id):
                discord_channel = bot.get_channel(discord_channel_id)
                if discord_channel:
                    await discord_channel.send(f"🎥 **{channel_name}** just posted a new video!\n{latest_video.link}")
                    
    conn.close()

@check_new_videos.before_loop
async def before_check():
    await bot.wait_until_ready()

# --- SLASH COMMANDS ---

@bot.tree.command(name="add", description="Add a YouTube channel (Use URL, @Handle, or Name)")
async def add_channel(interaction: discord.Interaction, channel_name: str):
    await interaction.response.defer()
    
    yt_channel_id, yt_channel_name = get_channel_info(channel_name)
    if not yt_channel_id:
        await interaction.followup.send(f"❌ Could not find a YouTube channel matching '{channel_name}'.")
        return
        
    conn = sqlite3.connect('channels.db')
    c = conn.cursor()
    
    # Check if already tracked
    c.execute("SELECT * FROM channels WHERE channel_id = ?", (yt_channel_id,))
    if c.fetchone():
        await interaction.followup.send(f"⚠️ **{yt_channel_name}** is already in the feed!")
        conn.close()
        return
        
    # Fetch recent videos to set the baseline and post the latest one
    feed_url = f"https://www.youtube.com/feeds/videos.xml?channel_id={yt_channel_id}"
    feed = feedparser.parse(feed_url)
    
    latest_video_id = feed.entries[0].yt_videoid if feed.entries else "NONE"
    
    c.execute("INSERT INTO channels (channel_id, channel_name, last_video_id, discord_channel_id) VALUES (?, ?, ?, ?)",
              (yt_channel_id, yt_channel_name, latest_video_id, interaction.channel_id))
    conn.commit()
    conn.close()
    
    msg = f"✅ Added **{yt_channel_name}** to the feed!"
    
    # Find the most recent non-short video to post right now
    posted = False
    for entry in feed.entries:
        if not await is_short(entry.yt_videoid):
            msg += f"\nHere is their latest video: {entry.link}"
            posted = True
            break
            
    if not posted:
        msg += "\n(Could not find a recent standard video to post, only shorts found)."
        
    await interaction.followup.send(msg)

@bot.tree.command(name="random", description="Get a random video from a channel or the whole feed")
async def random_video(interaction: discord.Interaction, channel_name: Optional[str] = None):
    await interaction.response.defer()
    
    if channel_name:
        yt_channel_id, yt_channel_name = get_channel_info(channel_name)
        if not yt_channel_id:
            await interaction.followup.send(f"❌ Could not find a YouTube channel matching '{channel_name}'.")
            return
    else:
        conn = sqlite3.connect('channels.db')
        c = conn.cursor()
        c.execute("SELECT channel_id, channel_name FROM channels")
        rows = c.fetchall()
        conn.close()
        
        if not rows:
            await interaction.followup.send("❌ There are no channels currently in the feed. Add some first!")
            return
            
        # Pick a random channel from our tracked list
        yt_channel_id, yt_channel_name = random.choice(rows)
        
    # Get the "Uploads" playlist ID (replace UC with UU)
    uploads_playlist_id = "UU" + yt_channel_id[2:]
    
    # Fetch a batch of recent videos (up to 50) to pick from to save API quota
    request = youtube.playlistItems().list(
        part="snippet",
        playlistId=uploads_playlist_id,
        maxResults=50
    )
    response = request.execute()
    
    if not response.get("items"):
        await interaction.followup.send(f"❌ **{yt_channel_name}** has no videos.")
        return
        
    # Shuffle and find a non-short video
    items = response["items"]
    random.shuffle(items)
    
    for item in items:
        vid_id = item["snippet"]["resourceId"]["videoId"]
        if not await is_short(vid_id):
            await interaction.followup.send(f"🎲 Random video from **{yt_channel_name}**:\nhttps://www.youtube.com/watch?v={vid_id}")
            return
            
    await interaction.followup.send(f"❌ Could not find a recent standard video for **{yt_channel_name}** (only shorts found in the latest 50 videos).")


# --- ADMIN ONLY COMMANDS ---
# default_permissions(administrator=True) hides these commands from non-admins automatically!

@bot.tree.command(name="list", description="Admin: List all tracked YouTube channels")
@app_commands.default_permissions(administrator=True)
async def auditxyz(interaction: discord.Interaction):
    conn = sqlite3.connect('channels.db')
    c = conn.cursor()
    c.execute("SELECT channel_name FROM channels")
    rows = c.fetchall()
    conn.close()
    
    if not rows:
        await interaction.response.send_message("The feed is currently empty.", ephemeral=True)
        return
        
    channel_list = "\n".join([f"• {row[0]}" for row in rows])
    await interaction.response.send_message(f"📊 **Tracked Channels:**\n{channel_list}", ephemeral=True)

@bot.tree.command(name="remove", description="Admin: Remove a YouTube channel from the feed")
@app_commands.default_permissions(administrator=True)
async def remove_channel(interaction: discord.Interaction, channel_name: str):
    await interaction.response.defer(ephemeral=True)
    
    conn = sqlite3.connect('channels.db')
    c = conn.cursor()
    
    # We do a LIKE query to make removing a bit forgiving on exact casing
    c.execute("SELECT channel_id, channel_name FROM channels WHERE channel_name LIKE ?", (f"%{channel_name}%",))
    row = c.fetchone()
    
    if not row:
        await interaction.followup.send(f"❌ Could not find '{channel_name}' in the database.")
        conn.close()
        return
        
    c.execute("DELETE FROM channels WHERE channel_id = ?", (row[0],))
    conn.commit()
    conn.close()
    
    await interaction.followup.send(f"🗑️ Removed **{row[1]}** from the feed.")

bot.run(DISCORD_TOKEN)
