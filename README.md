# botemail
import discord
from discord import app_commands
import requests
import uuid
import asyncio
import re

TOKEN = "MTM5MzA2Mzc0MjI0NTQzNzU1MQ.Ge3_G_.VWOA_xZIkQ87ZX9tFfQYDX8OYCbF4QaboPbiKo"
GUILD_ID = 1390962457375146035  # ID server của bạn

class Bot10p(discord.Client):
    def __init__(self):
        super().__init__(intents=discord.Intents.default())
        self.tree = app_commands.CommandTree(self)

    async def setup_hook(self):
        # Sync slash command cho server cụ thể (hiện ngay lập tức)
        guild = discord.Object(id=GUILD_ID)
        self.tree.copy_global_to(guild=guild)
        await self.tree.sync(guild=guild)
        print("✅ Slash commands synced.")

client = Bot10p()

def create_mail_tm_account():
    session = requests.Session()
    domain_res = session.get("https://api.mail.tm/domains")
    domain = domain_res.json()["hydra:member"][0]["domain"]

    email = f"{uuid.uuid4().hex[:10]}@{domain}"
    password = uuid.uuid4().hex
    data = {"address": email, "password": password}

    acc_res = session.post("https://api.mail.tm/accounts", json=data)
    if acc_res.status_code != 201:
        return None

    token_res = session.post("https://api.mail.tm/token", json=data)
    token = token_res.json()["token"]

    return {"email": email, "password": password, "token": token, "session": session}

async def check_inbox(mail_info, embed_msg):
    headers = {"Authorization": f"Bearer {mail_info['token']}"}
    seen_ids = set()
    while True:
        try:
            inbox = mail_info['session'].get(
                "https://api.mail.tm/messages", headers=headers
            ).json()["hydra:member"]

            for msg in inbox:
                if msg["id"] not in seen_ids:
                    seen_ids.add(msg["id"])
                    msg_detail = mail_info['session'].get(
                        f"https://api.mail.tm/messages/{msg['id']}", headers=headers
                    ).json()
                    subject = msg_detail.get("subject", "")
                    text = msg_detail.get("text", "")

                    # Chỉ lấy số
                    numbers = re.findall(r"\d+", subject + " " + text)
                    code = numbers[0] if numbers else "Không tìm thấy mã"

                    embed = embed_msg.embeds[0]
                    embed.set_field_at(1, name="📩 Thư của bạn:", value=f"```\n{code}\n```", inline=False)
                    await embed_msg.edit(embed=embed)
                    return
        except Exception as e:
            print("❌ Lỗi khi check inbox:", e)

        await asyncio.sleep(10)

@client.tree.command(name="bot10p", description="Tạo email tạm thời (.com)")
async def bot10p(interaction: discord.Interaction):
    mail_info = create_mail_tm_account()
    if not mail_info:
        await interaction.response.send_message("❌ Không thể tạo email. Thử lại sau.", ephemeral=True)
        return

    # Tag người dùng trong kênh
    await interaction.response.send_message(f"@{interaction.user.display_name} Email đã được gửi qua tin nhắn riêng!", allowed_mentions=discord.AllowedMentions(users=True))

    # Gửi email qua DM
    embed = discord.Embed(title="📧 Email của bạn c o n c ặ c ", color=0x00ffcc)
    embed.add_field(name="✉️ Email:", value=f"```\n{mail_info['email']}\n```", inline=False)
    embed.add_field(name="📩 Thư của bạn:", value="⏳ Đang chờ thư...", inline=False)

    try:
        dm = await interaction.user.create_dm()
        embed_msg = await dm.send(embed=embed)
        asyncio.create_task(check_inbox(mail_info, embed_msg))
    except discord.Forbidden:
        await interaction.followup.send("⚠️ Không thể gửi DM cho bạn. Hãy bật nhận tin nhắn từ server.")

client.run(TOKEN)
