# -const { 
  Client, 
  GatewayIntentBits, 
  ActionRowBuilder, 
  ButtonBuilder, 
  ButtonStyle, 
  EmbedBuilder 
} = require('discord.js');

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ],
});

// 出勤中のユーザーデータを一時保存するMap (ユーザーID : 出勤時刻のミリ秒)
const attendanceMap = new Map();

client.once('ready', () => {
  console.log(`${client.user.tag} が起動しました！`);
});

// 「!panel」と送信すると出退勤ボタンを設置
client.on('messageCreate', async (message) => {
  if (message.author.bot) return;

  if (message.content === '!panel') {
    const embed = new EmbedBuilder()
      .setTitle('🕒 出退勤パネル')
      .setDescription('下のボタンを押して出勤・退勤を記録してください。')
      .setColor(0x00ae86);

    const row = new ActionRowBuilder().addComponents(
      new ButtonBuilder()
        .setCustomId('clock_in')
        .setLabel('出勤')
        .setStyle(ButtonStyle.Success)
        .setEmoji('🟢'),
      new ButtonBuilder()
        .setCustomId('clock_out')
        .setLabel('退勤')
        .setStyle(ButtonStyle.Danger)
        .setEmoji('🔴')
    );

    await message.channel.send({ embeds: [embed], components: [row] });
  }
});

// ボタンが押されたときの処理
client.on('interactionCreate', async (interaction) => {
  if (!interaction.isButton()) return;

  const userId = interaction.user.id;
  const userName = interaction.member ? interaction.member.displayName : interaction.user.username;
  const now = new Date();
  const timeString = now.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' });

  // 【出勤ボタン】
  if (interaction.customId === 'clock_in') {
    if (attendanceMap.has(userId)) {
      return interaction.reply({
        content: '⚠️ すでに出勤中です！',
        ephemeral: true, // 押した本人にだけ表示
      });
    }

    attendanceMap.set(userId, now.getTime());

    await interaction.reply({
      content: `🟢 **${userName}** さんが **${timeString}** に出勤しました。`,
    });
  }

  // 【退勤ボタン】
  if (interaction.customId === 'clock_out') {
    if (!attendanceMap.has(userId)) {
      return interaction.reply({
        content: '⚠️ 出勤記録がありません。先に「出勤」ボタンを押してください。',
        ephemeral: true, // 押した本人にだけ表示
      });
    }

    const startTime = attendanceMap.get(userId);
    attendanceMap.delete(userId);

    // 勤務時間の計算
    const diffMs = now.getTime() - startTime;
    const totalMinutes = Math.floor(diffMs / (1000 * 60));
    const hours = Math.floor(totalMinutes / 60);
    const minutes = totalMinutes % 60;

    await interaction.reply({
      content: `🔴 **${userName}** さんが **${timeString}** に退勤しました！お疲れ様でした。\n⏱️ 勤務時間: **${hours}時間${minutes}分**`,
    });
  }
});

// ここに自分のBotのトークンを入れる
client.login('YOUR_BOT_TOKEN_HERE');
