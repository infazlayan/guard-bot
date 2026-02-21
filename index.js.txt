const { Client, GatewayIntentBits, AuditLogEvent, PermissionsBitField } = require("discord.js");

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.GuildModeration
  ]
});

const TOKEN = process.env.TOKEN;

// whitelist
const SAFE_USERS = [
  "1465454083665035409",
  "912804505320566834"
];

// ceza fonksiyonu
async function punish(guild, userId, reason) {
  try {
    if (SAFE_USERS.includes(userId)) return;

    const member = await guild.members.fetch(userId).catch(() => {});
    if (!member) return;

    await member.roles.set([]);
    await member.timeout(28 * 24 * 60 * 60 * 1000, reason);

    console.log(`${member.user.tag} cezalandırıldı → ${reason}`);
  } catch (err) {
    console.log("Ceza hatası:", err.message);
  }
}

client.on("ready", () => {
  console.log(`${client.user.tag} aktif`);
});

// Kanal silme guard
client.on("channelDelete", async (channel) => {
  const logs = await channel.guild.fetchAuditLogs({
    limit: 1,
    type: AuditLogEvent.ChannelDelete
  });

  const entry = logs.entries.first();
  if (!entry) return;

  punish(channel.guild, entry.executor.id, "Kanal sildi");
});

// Rol silme guard
client.on("roleDelete", async (role) => {
  const logs = await role.guild.fetchAuditLogs({
    limit: 1,
    type: AuditLogEvent.RoleDelete
  });

  const entry = logs.entries.first();
  if (!entry) return;

  punish(role.guild, entry.executor.id, "Rol sildi");
});

// Ban guard
client.on("guildBanAdd", async (ban) => {
  const logs = await ban.guild.fetchAuditLogs({
    limit: 1,
    type: AuditLogEvent.MemberBanAdd
  });

  const entry = logs.entries.first();
  if (!entry) return;

  punish(ban.guild, entry.executor.id, "Üye banladı");
});

// Kick guard
client.on("guildMemberRemove", async (member) => {
  const logs = await member.guild.fetchAuditLogs({
    limit: 1,
    type: AuditLogEvent.MemberKick
  });

  const entry = logs.entries.first();
  if (!entry) return;

  punish(member.guild, entry.executor.id, "Üye kickledi");
});

// Bot guard
client.on("guildMemberAdd", async (member) => {
  if (!member.user.bot) return;

  const logs = await member.guild.fetchAuditLogs({
    limit: 1,
    type: AuditLogEvent.BotAdd
  });

  const entry = logs.entries.first();
  if (!entry) return;

  const executor = entry.executor.id;

  if (SAFE_USERS.includes(executor)) return;

  await member.ban({ reason: "İzinsiz bot eklendi" });

  punish(member.guild, executor, "Bot ekledi");
});

client.login(TOKEN);