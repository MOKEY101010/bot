// index.js
const { Client, GatewayIntentBits, EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle, PermissionsBitField, ChannelType } = require('discord.js');

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ]
});

// Configurações
const CONFIG = {
  CANAL_MENSAGENS: '1461206809712001055',
  CATEGORIA_TICKETS: '1461176702226141367',
  CARGOS_STAFF: [
    '1461149686961275055',
    '1461139787577557123',
    '1461141649739616328',
    '1461147259084214363',
    '1461139246361612311'
  ],
  COR_EMBED: 0xFF0000, // Vermelho
  THUMBNAIL_URL: 'https://i.imgur.com/your-image.png' // Substitua pela sua imagem
};

// Armazenamento em memória
const filas = new Map(); // userId -> { modo, timestamp }
const tickets = new Map(); // channelId -> { criador, jogador2, modo, status }

// Função para criar o embed principal
function criarEmbedPrincipal() {
  return new EmbedBuilder()
    .setTitle('1v1 Mobile BELGA E-SPORTS')
    .addFields(
      { name: 'Modo', value: '1v1 Mobile', inline: true },
      { name: 'Valor', value: 'R$ 0,75', inline: true },
      { name: 'Jogadores', value: 'Gelo Infinito | @belgaesports', inline: false }
    )
    .setThumbnail(CONFIG.THUMBNAIL_URL)
    .setColor(CONFIG.COR_EMBED);
}

// Função para criar os botões principais
function criarBotoesPrincipais() {
  return new ActionRowBuilder()
    .addComponents(
      new ButtonBuilder()
        .setCustomId('gelo_infinito')
        .setLabel('Gelo Infinito')
        .setEmoji('🧊')
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId('gelo_normal')
        .setLabel('Gelo Normal')
        .setEmoji('❄️')
        .setStyle(ButtonStyle.Primary),
      new ButtonBuilder()
        .setCustomId('sair_fila')
        .setLabel('Sair')
        .setEmoji('🚪')
        .setStyle(ButtonStyle.Danger)
    );
}

// Função para enviar as 10 mensagens
async function enviarMensagensIniciais(canal) {
  const embed = criarEmbedPrincipal();
  const botoes = criarBotoesPrincipais();

  for (let i = 0; i < 10; i++) {
    await canal.send({ embeds: [embed], components: [botoes] });
  }
}

// Função para criar ticket
async function criarTicket(guild, usuario, modo) {
  const categoria = await guild.channels.fetch(CONFIG.CATEGORIA_TICKETS);
  const nomeTicket = modo === 'gelo_infinito' ? '1v1-gelo-infinito' : '1v1-gelo-normal';
  const modoTexto = modo === 'gelo_infinito' ? 'Gelo Infinito' : 'Gelo Normal';

  // Permissões iniciais
  const permissoes = [
    {
      id: guild.id,
      deny: [PermissionsBitField.Flags.ViewChannel]
    },
    {
      id: usuario.id,
      allow: [PermissionsBitField.Flags.ViewChannel, PermissionsBitField.Flags.SendMessages]
    }
  ];

  // Adicionar cargos de staff
  CONFIG.CARGOS_STAFF.forEach(cargoId => {
    permissoes.push({
      id: cargoId,
      allow: [PermissionsBitField.Flags.ViewChannel, PermissionsBitField.Flags.SendMessages]
    });
  });

  const ticket = await guild.channels.create({
    name: `${nomeTicket}-${usuario.username}`,
    type: ChannelType.GuildText,
    parent: categoria.id,
    permissionOverwrites: permissoes
  });

  // Criar embed do ticket
  const embedTicket = new EmbedBuilder()
    .setTitle('🎮 Ticket 1v1 Mobile')
    .addFields(
      { name: 'Modo', value: '1V1', inline: true },
      { name: 'Tipo', value: modoTexto, inline: true },
      { name: 'Jogador aguardando', value: usuario.tag, inline: false },
      { name: 'Status', value: '🕒 Aguardando adversário', inline: false }
    )
    .setColor(CONFIG.COR_EMBED)
    .setTimestamp();

  const botoesTicket = new ActionRowBuilder()
    .addComponents(
      new ButtonBuilder()
        .setCustomId('entrar_1v1')
        .setLabel('Entrar no 1V1')
        .setEmoji('➕')
        .setStyle(ButtonStyle.Success)
    );

  await ticket.send({ 
    content: `${usuario}, seu ticket foi criado!`,
    embeds: [embedTicket], 
    components: [botoesTicket] 
  });

  // Salvar informações do ticket
  tickets.set(ticket.id, {
    criador: usuario.id,
    jogador2: null,
    modo: modoTexto,
    status: 'aguardando'
  });

  return ticket;
}

// Função para atualizar permissões do ticket
async function atualizarPermissoesTicket(ticket, jogador2Id) {
  await ticket.permissionOverwrites.edit(jogador2Id, {
    ViewChannel: true,
    SendMessages: true
  });
}

// Função para criar botões de confirmação
function criarBotoesConfirmacao() {
  return new ActionRowBuilder()
    .addComponents(
      new ButtonBuilder()
        .setCustomId('confirmar_1v1')
        .setLabel('Confirmar')
        .setEmoji('✅')
        .setStyle(ButtonStyle.Success),
      new ButtonBuilder()
        .setCustomId('cancelar_1v1')
        .setLabel('Cancelar')
        .setEmoji('❌')
        .setStyle(ButtonStyle.Danger)
    );
}

// Verificar se usuário tem cargo de staff
function isStaff(member) {
  return CONFIG.CARGOS_STAFF.some(cargoId => member.roles.cache.has(cargoId));
}

// Event: Bot pronto
client.once('ready', async () => {
  console.log(`✅ Bot online como ${client.user.tag}`);
  
  // Opcional: Enviar mensagens ao iniciar (comente se não quiser que envie toda vez)
  // const canal = await client.channels.fetch(CONFIG.CANAL_MENSAGENS);
  // await enviarMensagensIniciais(canal);
});

// Event: Interações
client.on('interactionCreate', async interaction => {
  if (!interaction.isButton()) return;

  const { customId, user, guild, channel, member } = interaction;

  // Botão: Gelo Infinito ou Gelo Normal
  if (customId === 'gelo_infinito' || customId === 'gelo_normal') {
    // Verificar se usuário já está na fila
    if (filas.has(user.id)) {
      return interaction.reply({ 
        content: '❌ Você já está na fila!', 
        ephemeral: true 
      });
    }

    // Adicionar à fila
    filas.set(user.id, { modo: customId, timestamp: Date.now() });

    await interaction.reply({ 
      content: 'Você entrou na fila 🕒', 
      ephemeral: true 
    });

    // Criar ticket
    await criarTicket(guild, user, customId);
  }

  // Botão: Sair da fila
  if (customId === 'sair_fila') {
    if (!filas.has(user.id)) {
      return interaction.reply({ 
        content: '❌ Você não está na fila!', 
        ephemeral: true 
      });
    }

    filas.delete(user.id);
    
    await interaction.reply({ 
      content: '✅ Você saiu da fila!', 
      ephemeral: true 
    });
  }

  // Botão: Entrar no 1v1
  if (customId === 'entrar_1v1') {
    const ticketInfo = tickets.get(channel.id);

    if (!ticketInfo) {
      return interaction.reply({ 
        content: '❌ Erro ao encontrar informações do ticket!', 
        ephemeral: true 
      });
    }

    if (ticketInfo.criador === user.id) {
      return interaction.reply({ 
        content: '❌ Você não pode entrar no seu próprio ticket!', 
        ephemeral: true 
      });
    }

    if (ticketInfo.jogador2) {
      return interaction.reply({ 
        content: '❌ Este ticket já está completo!', 
        ephemeral: true 
      });
    }

    // Adicionar segundo jogador
    ticketInfo.jogador2 = user.id;
    ticketInfo.status = 'completo';
    tickets.set(channel.id, ticketInfo);

    // Atualizar permissões
    await atualizarPermissoesTicket(channel, user.id);

    // Atualizar embed
    const criador = await guild.members.fetch(ticketInfo.criador);
    const embedAtualizado = new EmbedBuilder()
      .setTitle('🎮 Ticket 1v1 Mobile')
      .addFields(
        { name: 'Modo', value: '1V1', inline: true },
        { name: 'Tipo', value: ticketInfo.modo, inline: true },
        { name: 'Jogador 1', value: criador.user.tag, inline: false },
        { name: 'Jogador 2', value: user.tag, inline: false },
        { name: 'Status', value: '✅ Partida pronta!', inline: false }
      )
      .setColor(CONFIG.COR_EMBED)
      .setTimestamp();

    const botoesConfirm = criarBotoesConfirmacao();

    await interaction.update({ 
      embeds: [embedAtualizado], 
      components: [botoesConfirm] 
    });

    await channel.send(`${user} entrou no 1v1! Boa sorte aos dois! 🎮`);
  }

  // Botão: Confirmar
  if (customId === 'confirmar_1v1') {
    const ticketInfo = tickets.get(channel.id);

    if (!ticketInfo) {
      return interaction.reply({ 
        content: '❌ Erro ao encontrar informações do ticket!', 
        ephemeral: true 
      });
    }

    // Verificar permissão
    const podConfirmar = ticketInfo.criador === user.id || isStaff(member);

    if (!podConfirmar) {
      return interaction.reply({ 
        content: '❌ Apenas o criador do ticket ou staff podem confirmar!', 
        ephemeral: true 
      });
    }

    await interaction.reply({ 
      content: '✅ Partida confirmada! Boa sorte! 🎮', 
      ephemeral: false 
    });

    // Remover da fila
    filas.delete(ticketInfo.criador);
    if (ticketInfo.jogador2) filas.delete(ticketInfo.jogador2);
  }

  // Botão: Cancelar
  if (customId === 'cancelar_1v1') {
    const ticketInfo = tickets.get(channel.id);

    if (!ticketInfo) {
      return interaction.reply({ 
        content: '❌ Erro ao encontrar informações do ticket!', 
        ephemeral: true 
      });
    }

    // Verificar permissão
    const podCancelar = ticketInfo.criador === user.id || isStaff(member);

    if (!podCancelar) {
      return interaction.reply({ 
        content: '❌ Apenas o criador do ticket ou staff podem cancelar!', 
        ephemeral: true 
      });
    }

    await interaction.reply({ 
      content: '❌ Ticket cancelado! O canal será deletado em 5 segundos...', 
      ephemeral: false 
    });

    // Remover da fila
    filas.delete(ticketInfo.criador);
    if (ticketInfo.jogador2) filas.delete(ticketInfo.jogador2);

    // Deletar ticket
    tickets.delete(channel.id);
    setTimeout(async () => {
      await channel.delete();
    }, 5000);
  }
});

// Login
client.login('MTQ2MzI5MTA2MjAyMDg3MDM2OA.Gx0ral.eiUh063y1Lyklumg-PVs63IOLiMB6RJIc9PD7M');
const express = require("express");
const app = express();

app.get("/", (req, res) => {
  res.send("BELGA E-SPORTS BOT ONLINE");
});

const PORT = process.env.PORT || 8000;
app.listen(PORT, () => {
  console.log("Servidor web ativo");
});

