🤖 YouTube Channel Tracker for Discord
This is a lightweight Discord bot designed to monitor YouTube channels and post new videos to a specific Discord channel in real-time. It includes a built-in "Shorts" filter to ensure your feed stays focused on standard video content.

✨ Features

Real-time Tracking: Polls YouTube RSS feeds every 15 minutes for updates.


Intelligent Search: Add channels by Name, Handle (@name), or URL.
+1


Shorts Filter: Uses a custom header-check to prevent YouTube Shorts from cluttering the feed.
+1


Randomizer: Pulls a random video from your tracked list—great for community discovery.


Admin Controls: Slash commands to audit or remove channels easily.

🚀 Getting Started
1. Prerequisites
A Discord Bot Token (via the Discord Developer Portal).

A YouTube Data API v3 Key (via the Google Cloud Console).

Python 3.8+ installed.

2. Installation
Clone the repository and install the required libraries:

Bash
pip install -r requirements.txt
3. Configuration
Open bot.py and replace the placeholders with your actual keys:

DISCORD_TOKEN = "your_token_here"

YOUTUBE_API_KEY = "your_api_key_here"

4. Running on a VPS (Droplet)
To keep the bot running 24/7 on a VPS like a DigitalOcean Droplet:

Upload your files to the server.

Use a process manager like PM2 or systemd to ensure the bot restarts if the server reboots.

Bash
pm2 start bot.py --interpreter python3
🛠️ Commands
/add [channel]: Add a new YouTube channel to the feed.

/random: Get a random video from the tracked list.

/auditxyz: (Admin Only) List all currently tracked channels.

/remove [channel]: (Admin Only) Remove a channel from the database.

🔒 Privacy Note
The bot creates a local channels.db file to store your tracking list. This file is included in the .gitignore to ensure your personal data is never shared publicly.
