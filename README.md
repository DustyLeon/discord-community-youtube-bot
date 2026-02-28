# community-youtube-discord
Community YouTube Feed for Discord Servers 
# Community YT for Discord

A simple Discord bot that lets a server track YouTube channels and automatically post new videos in a Discord channel.

It supports:
- adding YouTube channels by **URL**, **@handle**, **channel ID**, or **name**
- automatically posting newly uploaded videos
- skipping **YouTube Shorts**
- returning a **random video** from a tracked channel
- admin-only commands to **list** and **remove** tracked channels

## How it works

The bot stores tracked YouTube channels in a local SQLite database (`channels.db`).

Every few minutes, it checks each tracked channel's YouTube RSS feed and posts any newly detected **non-Short** video to the Discord channel where that YouTube channel was originally added.

## Requirements

Install the required libraries with:

```bash
pip install -r requirements.txt
