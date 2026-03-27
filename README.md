import nextcord
from nextcord.ext import commands
from nextcord.ui import Button, View, UserSelect
import asyncio
import time

# --- [ ตั้งค่าพื้นฐาน ] ---
TOKEN = "MTQ4NTY4NzE5NzkzMDYyMzA3OQ.GGFyfj.WHI05MXWKaDBZxrzxo36-KDJ-AfVTMoLXAo5Wg"
IMAGE_URL = "https://media.discordapp.net/attachments/1476920174069153795/1477293881212141649/1.gif?ex=69c683a4&is=69c53224&hm=50d4ed8ff9d4cc08d50eab1103cd657aaf9e336339423cdec66b44fd8c05bd5b&=&width=1221&height=863"
CATEGORY_NAME = "Tickets"
SS_ROLE_NAME = "『 SS 』สภานักเรียน""〈 TC/คุณครู/แอดมิน 〉" # ชื่อยศต้องตรงกับใน Discord เป๊ะๆ

intents = nextcord.Intents.default()
intents.message_content = True
intents.members = True

bot = commands.Bot(intents=intents)

# ===== 1. เมนูเลือกรายชื่อผู้ใช้ (Invite Select) =====
class InviteUserSelect(UserSelect):
    def __init__(self):
        super().__init__(
            placeholder="เลือกผู้ใช้ที่ต้องการอนุญาตให้เข้าห้องนี้ได้:",
            min_values=1,
            max_values=1
        )

    async def callback(self, interaction: nextcord.Interaction):
        target_user = self.values[0]
        # ปรับสิทธิ์ให้คนที่ถูกเลือกมองเห็นห้องนี้
        await interaction.channel.set_permissions(
            target_user, 
            view_channel=True, 
            send_messages=True, 
            read_message_history=True
        )
        await interaction.response.send_message(f"✅ เพิ่ม {target_user.mention} เข้าห้องเรียบร้อยโดย {interaction.user.mention}")

# ===== 2. View สำหรับในห้อง Ticket (Invite & Close) =====
class CloseTicketView(View):
    def __init__(self):
        super().__init__(timeout=None)

    @nextcord.ui.button(label="Invite", style=nextcord.ButtonStyle.green, emoji="👤", custom_id="inv_btn_v4")
    async def invite_button(self, button: Button, interaction: nextcord.Interaction):
        # เช็คสิทธิ์: เป็น Admin หรือ มียศ SS สภานักเรียน
        is_admin = interaction.user.guild_permissions.manage_channels
        has_ss_role = nextcord.utils.get(interaction.user.roles, name=SS_ROLE_NAME) is not None
        
        if is_admin or has_ss_role:
            # ถ้ามีสิทธิ์ ให้แสดงเมนูเลือกคนแบบภาพที่ 2
            view = View(timeout=60)
            view.add_item(InviteUserSelect())
            await interaction.response.send_message("เลือกผู้ใช้ที่ต้องการอนุญาตให้เข้าห้องนี้ได้:", view=view)
        else:
            await interaction.response.send_message(f"❌ เฉพาะแอดมินหรือผู้มียศ **{SS_ROLE_NAME}** เท่านั้นที่ใช้ปุ่มนี้ได้!", ephemeral=True)

    @nextcord.ui.button(label="Close Ticket", style=nextcord.ButtonStyle.red, emoji="🏹", custom_id="cls_btn_v4")
    async def close_button(self, button: Button, interaction: nextcord.Interaction):
        # เช็คสิทธิ์ Admin หรือ SS ก่อนปิด
        is_admin = interaction.user.guild_permissions.manage_channels
        has_ss_role = nextcord.utils.get(interaction.user.roles, name=SS_ROLE_NAME) is not None

        if is_admin or has_ss_role:
            await interaction.response.send_message("🛡️ ระบบจะทำการปิดห้องนี้ภายใน 5 วินาที...")
            await asyncio.sleep(5)
            await interaction.channel.delete()
        else:
            await interaction.response.send_message("❌ คุณไม่มีสิทธิ์ปิดห้องนี้!", ephemeral=True)

# ===== 3. View สำหรับหน้าแรก (สร้าง Ticket) =====
class TicketView(View):
    def __init__(self):
        super().__init__(timeout=None)

    async def create_tkt(self, interaction: nextcord.Interaction, tkt_type: str):
        await interaction.response.defer(ephemeral=True)
        guild = interaction.guild
        user = interaction.user
        
        # จัดการหมวดหมู่
        category = nextcord.utils.get(guild.categories, name=CATEGORY_NAME)
        if not category: 
            category = await guild.create_category(CATEGORY_NAME)
        
        ch_name = f"{tkt_type}-{user.name}".lower().replace(" ", "-")
        
        # ตั้งค่าสิทธิ์เริ่มต้น (คนเปิด + บอท)
        overwrites = {
            guild.default_role: nextcord.PermissionOverwrite(view_channel=False),
            user: nextcord.PermissionOverwrite(view_channel=True, send_messages=True, read_message_history=True),
            guild.me: nextcord.PermissionOverwrite(view_channel=True, send_messages=True)
        }
        
        # --- [ จุดที่ 2: ถ้าแจ้งความ ให้ยศ SS เห็นห้องทันที ] ---
        if tkt_type == "แจ้งความ":
            ss_role = nextcord.utils.get(guild.roles, name=SS_ROLE_NAME)
            if ss_role:
                overwrites[ss_role] = nextcord.PermissionOverwrite(view_channel=True, send_messages=True, read_message_history=True)

        # สร้างห้องจริง
        channel = await guild.create_text_channel(name=ch_name, category=category, overwrites=overwrites)
        
        # สร้าง Embed สวยๆ ตามรูปภาพที่ 1
        embed = nextcord.Embed(
            title=f"Ticket / {tkt_type}", 
            description="**ยินดีต้อนรับครับ**\nแจ้งเรื่องที่ต้องการให้ทีมงานช่วยเหลือได้เลย", 
            color=0x2ecc71
        )
        now = int(time.time())
        embed.add_field(name="📣 : คนเปิด", value=user.mention, inline=True)
        embed.add_field(name="🕒 : เวลาเปิด", value=f"<t:{now}:R>", inline=True)
        embed.set_author(name=user.display_name, icon_url=user.display_avatar.url)
        
        await channel.send(content=f"{user.mention} | ทีมงานได้รับเรื่องแล้ว!", embed=embed, view=CloseTicketView())
        await interaction.followup.send(f"สร้างห้องสำเร็จ: {channel.mention}", ephemeral=True)

    @nextcord.ui.button(label="แจ้งความ", style=nextcord.ButtonStyle.red, emoji="🚨", custom_id="btn_rep")
    async def report(self, b, i): await self.create_tkt(i, "แจ้งความ")

    @nextcord.ui.button(label="ซื้อยศ", style=nextcord.ButtonStyle.green, emoji="💎", custom_id="btn_buy")
    async def buy(self, b, i): await self.create_tkt(i, "ซื้อยศ")

    @nextcord.ui.button(label="ติดต่อแอดมิน", style=nextcord.ButtonStyle.blurple, emoji="🛠️", custom_id="btn_admin")
    async def admin(self, b, i): await self.create_tkt(i, "ติดต่อแอดมิน")

# ===== 4. เหตุการณ์และคำสั่ง =====
@bot.event
async def on_ready():
    bot.add_view(TicketView())
    bot.add_view(CloseTicketView())
    print(f'✅ บอทออนไลน์แล้ว: {bot.user}')

@bot.slash_command(name="setup_ticket", description="ตั้งค่าระบบ Ticket")
async def setup_ticket(interaction: nextcord.Interaction):
    # 1. สร้าง Embed ด้วยเนื้อหาที่คุณต้องการ (เช่น เปลี่ยนหัวข้อ เปลี่ยนข้อความ)
    embed = nextcord.Embed(
        title="ติดต่อสอบถาม / แจ้งเรื่อง", # แก้หัวข้อให้ตรงกับในภาพ
        description="""เลือกประเภทที่ต้องการติดต่อแอดมิน โดยคลิกปุ่มด้านล่าง
🍓 แจ้งความ = แจ้งความ/คดีทะเลาะ/ความไม่สงบภายในดิส
🥦 ซื้อยศ = ซื้อยศในดิส/โดเนท/ส่งหลักฐานกางเงิน
🍇 ติดต่อแอดมิน = แจ้งปัญหาต่างๆ/ติดต่อสอบถาม/เรียกแอดมิน""", # พิมพ์เนื้อหาใหม่ตามต้องการ
        color=0x2ecc71 # เปลี่ยนสี (ในรูปคือสีเขียวสด)
    )
    
    # 2. ใส่รูปภาพ (คุณต้องมีลิงก์รูปภาพของคุณเองนะ)
    # ใส่ลิงก์รูปภาพที่คุณอัปโหลดไว้ (Direct Link) ใน ""
    embed.set_image(url="file:///C:/Users/User/Downloads/1.gif") # ลิงก์รูปภาพในภาพตัวอย่าง
    # embed.set_image(url="วางลิงก์รูปภาพ GIF ของคุณที่นี่") # แก้ไขตรงนี้ด้วยลิงก์รูปภาพของคุณ!

    # 3. ใส่รูปภาพเล็กมุมบน (Author Icon)
    embed.set_author(name=f"{bot.user.display_name}", icon_url=bot.user.display_avatar.url)

    # 4. ส่ง Embed พร้อมปุ่ม (View) ไปยัง Discord
    await interaction.response.send_message(embed=embed, view=TicketView())

    embed.set_image(url=IMAGE_URL)
    await interaction.response.send_message(embed=embed, view=TicketView())

bot.run("MTQ4NTY4NzE5NzkzMDYyMzA3OQ.GGFyfj.WHI05MXWKaDBZxrzxo36-KDJ-AfVTMoLXAo5Wg")
