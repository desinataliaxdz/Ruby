import discord
from discord import app_commands
from discord.ext import commands
import sqlite3

# ============================================================
# SETTINGS
# ============================================================

GUILD_ID = 1542775353574039682

WIN_AP = 2
LOSE_AP = 1

OFFICER_ROLE = "Officers"

MAZE_TYPES = [
    "DW",
    "DS",
    "FC",
    "CA",
    "CI",
    "DC",
    "CD",
    "Boss"
]

# ============================================================
# DATABASE
# ============================================================

db = sqlite3.connect("guild.db")
cursor = db.cursor()

# ------------------------------------------------------------
# Member AP
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS members (
    guild_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    ap INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (guild_id, user_id)
)
""")

# ------------------------------------------------------------
# Guild Bank
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS bank (
    guild_id INTEGER PRIMARY KEY,
    gold INTEGER NOT NULL DEFAULT 0
)
""")

# ------------------------------------------------------------
# Guild Bank Items
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS bank_items (
    guild_id INTEGER NOT NULL,
    item_name TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (guild_id, item_name)
)
""")

# ------------------------------------------------------------
# AP Reports
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS ap_reports (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    guild_id INTEGER NOT NULL,
    reporter_id INTEGER NOT NULL,
    outcome TEXT NOT NULL,
    maze_type TEXT NOT NULL,
    ap_amount INTEGER NOT NULL,
    timestamp TEXT NOT NULL
)
""")

# ------------------------------------------------------------
# Members Included in AP Reports
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS ap_report_members (
    report_id INTEGER NOT NULL,
    user_id INTEGER NOT NULL,
    PRIMARY KEY (report_id, user_id)
)
""")

# ------------------------------------------------------------
# Item Prices
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS item_prices (
    guild_id INTEGER NOT NULL,
    item_name TEXT NOT NULL,
    ap_price INTEGER NOT NULL,
    PRIMARY KEY (guild_id, item_name)
)
""")

# ------------------------------------------------------------
# Withdrawal History
# ------------------------------------------------------------

cursor.execute("""
CREATE TABLE IF NOT EXISTS withdrawals (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    guild_id INTEGER NOT NULL,
    member_id INTEGER NOT NULL,
    officer_id INTEGER NOT NULL,
    item_name TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    ap_cost INTEGER NOT NULL,
    timestamp TEXT NOT NULL
)
""")

db.commit()


# ============================================================
# DATABASE FUNCTIONS
# ============================================================

def ensure_member(guild_id, user_id):
    cursor.execute("""
        INSERT OR IGNORE INTO members
        (guild_id, user_id, ap)
        VALUES (?, ?, 0)
    """, (guild_id, user_id))

    db.commit()


def get_ap(guild_id, user_id):
    ensure_member(guild_id, user_id)

    cursor.execute("""
        SELECT ap
        FROM members
        WHERE guild_id = ? AND user_id = ?
    """, (guild_id, user_id))

    result = cursor.fetchone()

    if result is None:
        return 0

    return result[0]


def change_ap(guild_id, user_id, amount):
    ensure_member(guild_id, user_id)

    cursor.execute("""
        UPDATE members
        SET ap = ap + ?
        WHERE guild_id = ? AND user_id = ?
    """, (amount, guild_id, user_id))

    db.commit()


# ============================================================
# OFFICER CHECK
# ============================================================

def is_officer(member):
    return any(
        role.name == OFFICER_ROLE
        for role in member.roles
    )


async def officer_only(interaction):
    if not isinstance(interaction.user, discord.Member):
        return False

    if not is_officer(interaction.user):
        await interaction.response.send_message(
            "❌ You need the **Officers** role to use this command.",
            ephemeral=True
        )
        return False

    return True


# ============================================================
# BOT
# ============================================================

class Pampy(commands.Bot):

    def __init__(self):
        intents = discord.Intents.default()

        # Required for checking voice channel members
        intents.voice_states = True

        super().__init__(
            command_prefix="!",
            intents=intents
        )

    async def setup_hook(self):

        # Sync ONLY to your specific Discord server.
        # This prevents duplicate commands.

        guild = discord.Object(id=GUILD_ID)

        await self.tree.sync(guild=guild)

        print(
            f"Slash commands synced to guild {GUILD_ID}"
        )

    async def on_ready(self):
        print("----------------------------------------")
        print(f"Logged in as: {self.user}")
        print(f"Bot ID: {self.user.id}")
        print(f"Connected to {len(self.guilds)} server(s)")
        print(f"Guild ID: {GUILD_ID}")
        print("----------------------------------------")


bot = Pampy()


# ============================================================
# /balance
# ============================================================

@bot.tree.command(
    name="balance",
    description="Check a member's AP balance",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    member="Member to check"
)
async def balance(
    interaction: discord.Interaction,
    member: discord.Member = None
):

    member = member or interaction.user

    ap = get_ap(
        interaction.guild.id,
        member.id
    )

    embed = discord.Embed(
        title="⭐ AP Balance",
        description=(
            f"**Member:** {member.mention}\n"
            f"**AP:** {ap}"
        )
    )

    await interaction.response.send_message(
        embed=embed
    )


# ============================================================
# /leaderboard
# ============================================================

@bot.tree.command(
    name="leaderboard",
    description="Show the AP leaderboard",
    guild=discord.Object(id=GUILD_ID)
)
async def leaderboard(
    interaction: discord.Interaction
):

    cursor.execute("""
        SELECT user_id, ap
        FROM members
        WHERE guild_id = ?
        ORDER BY ap DESC
        LIMIT 20
    """, (interaction.guild.id,))

    rows = cursor.fetchall()

    if not rows:
        await interaction.response.send_message(
            "No AP records yet."
        )
        return

    lines = []

    for position, (user_id, ap) in enumerate(
        rows,
        start=1
    ):

        member = interaction.guild.get_member(user_id)

        if member is None:
            try:
                member = await interaction.guild.fetch_member(
                    user_id
                )

            except (
                discord.NotFound,
                discord.HTTPException
            ):
                member = None

        if member:
            name = member.mention
        else:
            name = f"Unknown User ({user_id})"

        lines.append(
            f"**{position}.** {name} — **{ap} AP**"
        )

    embed = discord.Embed(
        title="🏆 AP Leaderboard",
        description="\n".join(lines)
    )

    await interaction.response.send_message(
        embed=embed,
        allowed_mentions=discord.AllowedMentions(
            users=True,
            roles=False,
            everyone=False
        )
    )


# ============================================================
# /add
# ============================================================

@bot.tree.command(
    name="add",
    description="Add AP to a member",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    member="Member receiving AP",
    amount="Amount of AP"
)
async def add(
    interaction: discord.Interaction,
    member: discord.Member,
    amount: int
):

    if not await officer_only(interaction):
        return

    if amount <= 0:
        await interaction.response.send_message(
            "❌ Amount must be greater than 0.",
            ephemeral=True
        )
        return

    change_ap(
        interaction.guild.id,
        member.id,
        amount
    )

    new_balance = get_ap(
        interaction.guild.id,
        member.id
    )

    await interaction.response.send_message(
        f"✅ Added **{amount} AP** to {member.mention}.\n"
        f"⭐ New balance: **{new_balance} AP**"
    )


# ============================================================
# /remove
# ============================================================

@bot.tree.command(
    name="remove",
    description="Remove AP from a member",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    member="Member losing AP",
    amount="Amount of AP"
)
async def remove(
    interaction: discord.Interaction,
    member: discord.Member,
    amount: int
):

    if not await officer_only(interaction):
        return

    if amount <= 0:
        await interaction.response.send_message(
            "❌ Amount must be greater than 0.",
            ephemeral=True
        )
        return

    current = get_ap(
        interaction.guild.id,
        member.id
    )

    if amount > current:
        await interaction.response.send_message(
            f"❌ {member.mention} only has **{current} AP**.",
            ephemeral=True
        )
        return

    change_ap(
        interaction.guild.id,
        member.id,
        -amount
    )

    new_balance = get_ap(
        interaction.guild.id,
        member.id
    )

    await interaction.response.send_message(
        f"✅ Removed **{amount} AP** from {member.mention}.\n"
        f"⭐ New balance: **{new_balance} AP**"
    )


# ============================================================
# /transfer
# ============================================================

@bot.tree.command(
    name="transfer",
    description="Transfer AP from one member to another",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    from_member="Member losing AP",
    to_member="Member receiving AP",
    quantity="Amount of AP to transfer"
)
async def transfer(
    interaction: discord.Interaction,
    from_member: discord.Member,
    to_member: discord.Member,
    quantity: int
):

    # --------------------------------------------------------
    # Officer check
    # --------------------------------------------------------

    if not await officer_only(interaction):
        return

    # --------------------------------------------------------
    # Validate quantity
    # --------------------------------------------------------

    if quantity <= 0:

        await interaction.response.send_message(
            "❌ Quantity must be greater than 0.",
            ephemeral=True
        )

        return

    # --------------------------------------------------------
    # Prevent transferring to the same person
    # --------------------------------------------------------

    if from_member.id == to_member.id:

        await interaction.response.send_message(
            "❌ The **From** and **To** members must be different.",
            ephemeral=True
        )

        return

    # --------------------------------------------------------
    # Check sender's AP
    # --------------------------------------------------------

    from_balance = get_ap(
        interaction.guild.id,
        from_member.id
    )

    if quantity > from_balance:

        await interaction.response.send_message(
            f"❌ {from_member.mention} does not have enough AP.\n\n"
            f"⭐ Current AP: **{from_balance} AP**\n"
            f"💰 Requested: **{quantity} AP**",
            ephemeral=True
        )

        return

    # --------------------------------------------------------
    # Perform transfer
    # --------------------------------------------------------

    change_ap(
        interaction.guild.id,
        from_member.id,
        -quantity
    )

    change_ap(
        interaction.guild.id,
        to_member.id,
        quantity
    )

    # --------------------------------------------------------
    # Get new balances
    # --------------------------------------------------------

    new_from_balance = get_ap(
        interaction.guild.id,
        from_member.id
    )

    new_to_balance = get_ap(
        interaction.guild.id,
        to_member.id
    )

    # --------------------------------------------------------
    # Confirmation
    # --------------------------------------------------------

    embed = discord.Embed(
        title="🔄 AP Transfer",
        description=(
            f"**From:** {from_member.mention}\n"
            f"**To:** {to_member.mention}\n"
            f"**Amount:** **{quantity} AP**\n\n"
            f"⭐ **{from_member.display_name}:** "
            f"{new_from_balance} AP\n"
            f"⭐ **{to_member.display_name}:** "
            f"{new_to_balance} AP\n\n"
            f"🛡️ **Approved by:** "
            f"{interaction.user.mention}"
        )
    )

    await interaction.response.send_message(
        embed=embed,
        allowed_mentions=discord.AllowedMentions(
            users=True,
            roles=False,
            everyone=False
        )
    )

# ============================================================
# /report
# ============================================================

@bot.tree.command(
    name="report",
    description="Report a maze and award AP to VC members",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    outcome="Win or Lose",
    maze="Maze type"
)
@app_commands.choices(
    outcome=[
        app_commands.Choice(
            name="Win",
            value="win"
        ),
        app_commands.Choice(
            name="Lose",
            value="lose"
        )
    ],
    maze=[
        app_commands.Choice(name="DW", value="DW"),
        app_commands.Choice(name="DS", value="DS"),
        app_commands.Choice(name="FC", value="FC"),
        app_commands.Choice(name="CA", value="CA"),
        app_commands.Choice(name="CI", value="CI"),
        app_commands.Choice(name="DC", value="DC"),
        app_commands.Choice(name="CD", value="CD"),
        app_commands.Choice(name="Boss", value="Boss")
    ]
)
async def report(
    interaction: discord.Interaction,
    outcome: app_commands.Choice[str],
    maze: app_commands.Choice[str]
):

    if not await officer_only(interaction):
        return

    if interaction.user.voice is None:
        await interaction.response.send_message(
            "❌ You must be in a voice channel.",
            ephemeral=True
        )
        return

    channel = interaction.user.voice.channel

    participants = [
        member
        for member in channel.members
        if not member.bot
    ]

    if not participants:
        await interaction.response.send_message(
            "❌ No members found in the voice channel.",
            ephemeral=True
        )
        return

    if outcome.value == "win":
        amount = WIN_AP
        emoji = "🟢"
    else:
        amount = LOSE_AP
        emoji = "🔴"

    timestamp = discord.utils.utcnow().isoformat()

    cursor.execute("""
        INSERT INTO ap_reports
        (
            guild_id,
            reporter_id,
            outcome,
            maze_type,
            ap_amount,
            timestamp
        )
        VALUES (?, ?, ?, ?, ?, ?)
    """, (
        interaction.guild.id,
        interaction.user.id,
        outcome.value,
        maze.value,
        amount,
        timestamp
    ))

    report_id = cursor.lastrowid

    for member in participants:

        ensure_member(
            interaction.guild.id,
            member.id
        )

        change_ap(
            interaction.guild.id,
            member.id,
            amount
        )

        cursor.execute("""
            INSERT INTO ap_report_members
            (report_id, user_id)
            VALUES (?, ?)
        """, (
            report_id,
            member.id
        ))

    db.commit()

    members_text = " ".join(
        member.mention
        for member in participants
    )

    embed = discord.Embed(
        title="📝 AP Report",
        description=(
            f"**Maze:** {maze.value}\n"
            f"**Outcome:** {emoji} {outcome.name}\n"
            f"**AP Awarded:** +{amount} AP each\n\n"
            f"**Participants:**\n"
            f"{members_text}"
        )
    )

    await interaction.response.send_message(
        content=members_text,
        embed=embed,
        allowed_mentions=discord.AllowedMentions(
            users=True,
            roles=False,
            everyone=False
        )
    )


# ============================================================
# ITEM PRICE GROUP
# ============================================================

itemprice_group = app_commands.Group(
    name="itemprice",
    description="Manage AP prices for items"
)


# ============================================================
# /itemprice set
# ============================================================

@itemprice_group.command(
    name="set",
    description="Set an item's AP price"
)
@app_commands.describe(
    item="Item name",
    price="AP price per item"
)
async def itemprice_set(
    interaction: discord.Interaction,
    item: str,
    price: int
):

    if not await officer_only(interaction):
        return

    item = item.strip()

    if not item:
        await interaction.response.send_message(
            "❌ Item name cannot be empty.",
            ephemeral=True
        )
        return

    if price <= 0:
        await interaction.response.send_message(
            "❌ Price must be greater than 0 AP.",
            ephemeral=True
        )
        return

    cursor.execute("""
        INSERT INTO item_prices
        (
            guild_id,
            item_name,
            ap_price
        )
        VALUES (?, ?, ?)
        ON CONFLICT(guild_id, item_name)
        DO UPDATE SET ap_price = excluded.ap_price
    """, (
        interaction.guild.id,
        item,
        price
    ))

    db.commit()

    await interaction.response.send_message(
        f"✅ **{item}** now costs **{price} AP**."
    )


# ============================================================
# /itemprice list
# ============================================================

@itemprice_group.command(
    name="list",
    description="List all item AP prices"
)
async def itemprice_list(
    interaction: discord.Interaction
):

    cursor.execute("""
        SELECT item_name, ap_price
        FROM item_prices
        WHERE guild_id = ?
        ORDER BY item_name COLLATE NOCASE
    """, (interaction.guild.id,))

    rows = cursor.fetchall()

    if not rows:
        await interaction.response.send_message(
            "📦 No item prices have been configured yet."
        )
        return

    lines = []

    for item, price in rows:
        lines.append(
            f"📦 **{item}** — **{price} AP**"
        )

    embed = discord.Embed(
        title="💰 Item AP Prices",
        description="\n".join(lines)
    )

    await interaction.response.send_message(
        embed=embed
    )


# ============================================================
# /itemprice remove
# ============================================================

@itemprice_group.command(
    name="remove",
    description="Remove an item's AP price"
)
@app_commands.describe(
    item="Item name"
)
async def itemprice_remove(
    interaction: discord.Interaction,
    item: str
):

    if not await officer_only(interaction):
        return

    item = item.strip()

    cursor.execute("""
        SELECT ap_price
        FROM item_prices
        WHERE guild_id = ?
        AND item_name = ?
    """, (
        interaction.guild.id,
        item
    ))

    result = cursor.fetchone()

    if result is None:
        await interaction.response.send_message(
            f"❌ No price exists for **{item}**.",
            ephemeral=True
        )
        return

    cursor.execute("""
        DELETE FROM item_prices
        WHERE guild_id = ?
        AND item_name = ?
    """, (
        interaction.guild.id,
        item
    ))

    db.commit()

    await interaction.response.send_message(
        f"✅ Removed the AP price for **{item}**."
    )


# ============================================================
# ADD ITEMPRICE GROUP TO BOT
# ============================================================

bot.tree.add_command(
    itemprice_group,
    guild=discord.Object(id=GUILD_ID)
)


# ============================================================
# /withdraw
# ============================================================

@bot.tree.command(
    name="withdraw",
    description="Withdraw an item for a member using AP",
    guild=discord.Object(id=GUILD_ID)
)
@app_commands.describe(
    member="Member receiving the item",
    item="Item being withdrawn",
    quantity="Quantity"
)
async def withdraw(
    interaction: discord.Interaction,
    member: discord.Member,
    item: str,
    quantity: int
):

    if not await officer_only(interaction):
        return

    item = item.strip()

    # --------------------------------------------------------
    # Validate quantity
    # --------------------------------------------------------

    if quantity <= 0:
        await interaction.response.send_message(
            "❌ Quantity must be greater than 0.",
            ephemeral=True
        )
        return

    # --------------------------------------------------------
    # Find item price
    # --------------------------------------------------------

    cursor.execute("""
        SELECT ap_price
        FROM item_prices
        WHERE guild_id = ?
        AND item_name = ?
    """, (
        interaction.guild.id,
        item
    ))

    price_result = cursor.fetchone()

    if price_result is None:
        await interaction.response.send_message(
            f"❌ **{item}** does not have an AP price configured.\n\n"
            f"Use `/itemprice set` first.",
            ephemeral=True
        )
        return

    price_per_item = price_result[0]

    # --------------------------------------------------------
    # CHECK BANK INVENTORY
    # --------------------------------------------------------

    cursor.execute("""
        SELECT quantity
        FROM bank_items
        WHERE guild_id = ?
        AND item_name = ?
    """, (
        interaction.guild.id,
        item
    ))

    bank_result = cursor.fetchone()

    if bank_result is None:
        await interaction.response.send_message(
            f"❌ **{item}** is not currently in the guild bank.",
            ephemeral=True
        )
        return

    bank_quantity = bank_result[0]

    if quantity > bank_quantity:
        await interaction.response.send_message(
            f"❌ Not enough **{item}** in the guild bank.\n\n"
            f"🏦 Bank has: **{bank_quantity}**\n"
            f"📦 Requested: **{quantity}**",
            ephemeral=True
        )
        return

    # --------------------------------------------------------
    # Calculate AP cost
    # --------------------------------------------------------

    total_cost = price_per_item * quantity

    current_ap = get_ap(
        interaction.guild.id,
        member.id
    )

    if total_cost > current_ap:
        await interaction.response.send_message(
            f"❌ {member.mention} does not have enough AP.\n\n"
            f"⭐ Current AP: **{current_ap} AP**\n"
            f"💰 Required: **{total_cost} AP**",
            ephemeral=True
        )
        return

    # --------------------------------------------------------
    # EVERYTHING IS VALID
    #
    # Now:
    # 1. Deduct AP
    # 2. Deduct item from bank
    # 3. Record withdrawal
    # --------------------------------------------------------

    # Deduct AP
    change_ap(
        interaction.guild.id,
        member.id,
        -total_cost
    )

    # Calculate remaining bank quantity
    new_bank_quantity = bank_quantity - quantity

    if new_bank_quantity == 0:

        # Delete item completely if quantity reaches zero
        cursor.execute("""
            DELETE FROM bank_items
            WHERE guild_id = ?
            AND item_name = ?
        """, (
            interaction.guild.id,
            item
        ))

    else:

        # Otherwise reduce quantity
        cursor.execute("""
            UPDATE bank_items
            SET quantity = ?
            WHERE guild_id = ?
            AND item_name = ?
        """, (
            new_bank_quantity,
            interaction.guild.id,
            item
        ))

    # Get new AP balance
    new_balance = get_ap(
        interaction.guild.id,
        member.id
    )

    # --------------------------------------------------------
    # Record withdrawal
    # --------------------------------------------------------

    timestamp = discord.utils.utcnow().isoformat()

    cursor.execute("""
        INSERT INTO withdrawals
        (
            guild_id,
            member_id,
            officer_id,
            item_name,
            quantity,
            ap_cost,
            timestamp
        )
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        interaction.guild.id,
        member.id,
        interaction.user.id,
        item,
        quantity,
        total_cost,
        timestamp
    ))

    db.commit()

    # --------------------------------------------------------
    # Success message
    # --------------------------------------------------------

    embed = discord.Embed(
        title="💸 Payday Withdrawal",
        description=(
            f"**Member:** {member.mention}\n"
            f"**Item:** {item}\n"
            f"**Quantity:** {quantity}\n"
            f"**Price Each:** {price_per_item} AP\n"
            f"**Total Cost:** {total_cost} AP\n\n"
            f"⭐ **Remaining AP:** {new_balance}\n"
            f"🏦 **Bank Remaining:** {new_bank_quantity}\n\n"
            f"🛡️ **Approved by:** {interaction.user.mention}"
        )
    )

    await interaction.response.send_message(
        content=member.mention,
        embed=embed,
        allowed_mentions=discord.AllowedMentions(
            users=True,
            roles=False,
            everyone=False
        )
    )


# ============================================================
# BANK GROUP
# ============================================================

bank_group = app_commands.Group(
    name="bank",
    description="Guild bank commands"
)


# ============================================================
# /bank view
# ============================================================

@bank_group.command(
    name="view",
    description="View the guild bank"
)
async def bank_view(
    interaction: discord.Interaction
):

    if not await officer_only(interaction):
        return

    cursor.execute("""
        INSERT OR IGNORE INTO bank
        (guild_id, gold)
        VALUES (?, 0)
    """, (interaction.guild.id,))

    cursor.execute("""
        SELECT gold
        FROM bank
        WHERE guild_id = ?
    """, (interaction.guild.id,))

    result = cursor.fetchone()

    gold = result[0] if result else 0

    cursor.execute("""
        SELECT item_name, quantity
        FROM bank_items
        WHERE guild_id = ?
        AND quantity > 0
        ORDER BY item_name COLLATE NOCASE
    """, (interaction.guild.id,))

    items = cursor.fetchall()

    if items:
        item_text = "\n".join(
            f"📦 {name} × **{quantity}**"
            for name, quantity in items
        )
    else:
        item_text = "No items."

    embed = discord.Embed(
        title="🏦 Guild Bank",
        description=(
            f"💰 **Gold:** {gold:,}\n\n"
            f"**Items:**\n{item_text}"
        )
    )

    await interaction.response.send_message(
        embed=embed
    )


# ============================================================
# BANK GOLD GROUP
# ============================================================

gold_group = app_commands.Group(
    name="gold",
    description="Guild bank gold"
)


# ============================================================
# /bank gold add
# ============================================================

@gold_group.command(
    name="add",
    description="Add gold to the guild bank"
)
@app_commands.describe(
    amount="Amount of gold"
)
async def gold_add(
    interaction: discord.Interaction,
    amount: int
):

    if not await officer_only(interaction):
        return

    if amount <= 0:
        await interaction.response.send_message(
            "❌ Amount must be greater than 0.",
            ephemeral=True
        )
        return

    cursor.execute("""
        INSERT OR IGNORE INTO bank
        (guild_id, gold)
        VALUES (?, 0)
    """, (interaction.guild.id,))

    cursor.execute("""
        UPDATE bank
        SET gold = gold + ?
        WHERE guild_id = ?
    """, (
        amount,
        interaction.guild.id
    ))

    db.commit()

    await interaction.response.send_message(
        f"✅ Added **{amount:,} gold** to the guild bank."
    )


# ============================================================
# /bank gold remove
# ============================================================

@gold_group.command(
    name="remove",
    description="Remove gold from the guild bank"
)
@app_commands.describe(
    amount="Amount of gold"
)
async def gold_remove(
    interaction: discord.Interaction,
    amount: int
):

    if not await officer_only(interaction):
        return

    if amount <= 0:
        await interaction.response.send_message(
            "❌ Amount must be greater than 0.",
            ephemeral=True
        )
        return

    cursor.execute("""
        INSERT OR IGNORE INTO bank
        (guild_id, gold)
        VALUES (?, 0)
    """, (interaction.guild.id,))

    cursor.execute("""
        SELECT gold
        FROM bank
        WHERE guild_id = ?
    """, (interaction.guild.id,))

    result = cursor.fetchone()

    gold = result[0] if result else 0

    if amount > gold:
        await interaction.response.send_message(
            f"❌ Bank only has **{gold:,} gold**.",
            ephemeral=True
        )
        return

    cursor.execute("""
        UPDATE bank
        SET gold = gold - ?
        WHERE guild_id = ?
    """, (
        amount,
        interaction.guild.id
    ))

    db.commit()

    await interaction.response.send_message(
        f"✅ Removed **{amount:,} gold** from the guild bank."
    )


# ============================================================
# ADD GOLD GROUP TO BANK
# ============================================================

bank_group.add_command(gold_group)


# ============================================================
# BANK ITEM GROUP
# ============================================================

item_group = app_commands.Group(
    name="item",
    description="Guild bank items"
)


# ============================================================
# /bank item add
# ============================================================

@item_group.command(
    name="add",
    description="Add an item to the guild bank"
)
@app_commands.describe(
    item="Item name",
    quantity="Quantity"
)
async def item_add(
    interaction: discord.Interaction,
    item: str,
    quantity: int
):

    if not await officer_only(interaction):
        return

    item = item.strip()

    if not item:
        await interaction.response.send_message(
            "❌ Item name cannot be empty.",
            ephemeral=True
        )
        return

    if quantity <= 0:
        await interaction.response.send_message(
            "❌ Quantity must be greater than 0.",
            ephemeral=True
        )
        return

    cursor.execute("""
        INSERT INTO bank_items
        (
            guild_id,
            item_name,
            quantity
        )
        VALUES (?, ?, ?)
        ON CONFLICT(guild_id, item_name)
        DO UPDATE SET quantity =
            quantity + excluded.quantity
    """, (
        interaction.guild.id,
        item,
        quantity
    ))

    db.commit()

    # Show updated quantity
    cursor.execute("""
        SELECT quantity
        FROM bank_items
        WHERE guild_id = ?
        AND item_name = ?
    """, (
        interaction.guild.id,
        item
    ))

    result = cursor.fetchone()

    new_quantity = result[0] if result else quantity

    await interaction.response.send_message(
        f"✅ Added **{quantity} × {item}** "
        f"to the guild bank.\n"
        f"🏦 Bank now has **{new_quantity} × {item}**."
    )


# ============================================================
# /bank item remove
# ============================================================

@item_group.command(
    name="remove",
    description="Remove an item from the guild bank"
)
@app_commands.describe(
    item="Item name",
    quantity="Quantity"
)
async def item_remove(
    interaction: discord.Interaction,
    item: str,
    quantity: int
):

    if not await officer_only(interaction):
        return

    item = item.strip()

    if quantity <= 0:
        await interaction.response.send_message(
            "❌ Quantity must be greater than 0.",
            ephemeral=True
        )
        return

    cursor.execute("""
        SELECT quantity
        FROM bank_items
        WHERE guild_id = ?
        AND item_name = ?
    """, (
        interaction.guild.id,
        item
    ))

    result = cursor.fetchone()

    if result is None:
        await interaction.response.send_message(
            f"❌ **{item}** isn't in the bank.",
            ephemeral=True
        )
        return

    current = result[0]

    if quantity > current:
        await interaction.response.send_message(
            f"❌ Bank only has "
            f"**{current} × {item}**.",
            ephemeral=True
        )
        return

    new_quantity = current - quantity

    if new_quantity == 0:

        cursor.execute("""
            DELETE FROM bank_items
            WHERE guild_id = ?
            AND item_name = ?
        """, (
            interaction.guild.id,
            item
        ))

    else:

        cursor.execute("""
            UPDATE bank_items
            SET quantity = ?
            WHERE guild_id = ?
            AND item_name = ?
        """, (
            new_quantity,
            interaction.guild.id,
            item
        ))

    db.commit()

    await interaction.response.send_message(
        f"✅ Removed **{quantity} × {item}** "
        f"from the guild bank.\n"
        f"🏦 Remaining: **{new_quantity} × {item}**."
    )


# ============================================================
# ADD ITEM GROUP TO BANK
# ============================================================

bank_group.add_command(item_group)


# ============================================================
# ADD BANK GROUP TO BOT
# ============================================================

bot.tree.add_command(
    bank_group,
    guild=discord.Object(id=GUILD_ID)
)


# ============================================================
# TOKEN
# ============================================================

import os

TOKEN = os.getenv("DISCORD_TOKEN")


# ============================================================
# RUN BOT
# ============================================================

bot.run(TOKEN)