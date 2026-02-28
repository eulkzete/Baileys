<h1 align='center'><img alt="Baileys logo" src="https://raw.githubusercontent.com/WhiskeySockets/Baileys/refs/heads/master/Media/logo.png" height="75"/></h1>

<div align='center'>Baileys é uma biblioteca TypeScript baseada em WebSockets para interagir diretamente com a API Web do WhatsApp.</div>

<br/>

> [!CAUTION]
> **AVISO DE BREAKING CHANGE**
>
> A partir da versão `7.0.0`, múltiplas mudanças incompatíveis foram introduzidas na biblioteca.
>
> Consulte o guia de migração em: https://whiskey.so/migrate-latest

---

## 📋 Índice

- [Por que Baileys?](#-por-que-baileys)
- [Instalação](#-instalação)
- [Conectando uma Conta](#-conectando-uma-conta)
  - [Via QR Code](#via-qr-code)
  - [Via Código de Pareamento](#via-código-de-pareamento)
  - [Recebendo Histórico Completo](#recebendo-histórico-completo)
- [Configurações Importantes do Socket](#-configurações-importantes-do-socket)
- [Salvando e Restaurando Sessão](#-salvando-e-restaurando-sessão)
- [Lidando com Eventos](#-lidando-com-eventos)
  - [Exemplo Básico](#exemplo-básico)
  - [Descriptografar Votos de Enquete](#descriptografar-votos-de-enquete)
  - [Resumo de Eventos na Primeira Conexão](#resumo-de-eventos-na-primeira-conexão)
- [Implementando um Data Store](#-implementando-um-data-store)
- [Entendendo os IDs do WhatsApp](#-entendendo-os-ids-do-whatsapp)
- [Funções Utilitárias](#-funções-utilitárias)
- [Enviando Mensagens](#-enviando-mensagens)
  - [Mensagens de Texto e Não-Mídia](#mensagens-de-texto-e-não-mídia)
  - [Mensagens com Prévia de Link](#mensagens-com-prévia-de-link)
  - [Mensagens de Mídia](#mensagens-de-mídia)
- [Modificando Mensagens](#-modificando-mensagens)
- [Manipulando Mídia](#-manipulando-mídia)
- [Rejeitar Chamada](#-rejeitar-chamada)
- [Estados no Chat](#-estados-no-chat)
- [Modificando Chats](#-modificando-chats)
- [Consultas de Usuário](#-consultas-de-usuário)
- [Alterando Perfil](#-alterando-perfil)
- [Grupos](#-grupos)
- [Privacidade](#-privacidade)
- [Listas de Transmissão e Stories](#-listas-de-transmissão-e-stories)
- [Funcionalidades Customizadas](#-funcionalidades-customizadas)
- [Links Úteis](#-links-úteis)
- [Aviso Legal](#-aviso-legal)
- [Licença](#-licença)

---

## ⚡ Por que Baileys?

- **Sem Selenium ou Chromium** — o Baileys se comunica diretamente com o WhatsApp via WebSocket, economizando mais de **meio gigabyte de RAM**.
- Suporte completo à **API Multi-Device** do WhatsApp.
- 100% TypeScript, com tipagem completa para autocompletar em editores como VS Code.

> [!IMPORTANT]
> O repositório original foi removido pelo autor original. Este é o único repositório oficial, mantido pela comunidade.
> **Entre no Discord: [discord.gg/WeJM5FP9GG](https://discord.gg/WeJM5FP9GG)**

---

## 📦 Instalação

**Versão estável:**
```bash
yarn add @whiskeysockets/baileys
```

**Versão edge** (sem garantia de estabilidade, mas com as últimas correções e features):
```bash
yarn add github:WhiskeySockets/Baileys
```

**Importando no seu projeto:**
```ts
import makeWASocket from '@whiskeysockets/baileys'
```

---

## 🔌 Conectando uma Conta

O WhatsApp oferece uma API multi-device que permite ao Baileys autenticar-se como um segundo cliente, seja por **QR Code** ou **Código de Pareamento**.

> [!TIP]
> Veja todas as configurações disponíveis do socket [aqui](https://baileys.whiskeysockets.io/types/SocketConfig.html).

### Via QR Code

> [!TIP]
> Você pode customizar o nome do browser com a constante `Browser`. Veja as opções disponíveis [aqui](https://baileys.whiskeysockets.io/types/BrowsersMap.html).

```ts
import makeWASocket, { Browsers } from '@whiskeysockets/baileys'

const sock = makeWASocket({
    browser: Browsers.ubuntu('Meu App'),
})
```

Quando a conexão for bem-sucedida, o QR Code será gerado — implemente a exibição dele da forma que preferir (terminal, imagem, etc.) escutando o evento `connection.update`.

### Via Código de Pareamento

> [!IMPORTANT]
> O Código de Pareamento **não é** a API Mobile. É um método alternativo ao QR Code para conectar o WhatsApp Web, permitindo apenas um dispositivo conectado por vez. Saiba mais [aqui](https://faq.whatsapp.com/1324084875126592/?cms_platform=web).

O número de telefone **não pode** conter `+`, `()` ou `-` — use apenas dígitos, incluindo o código do país.

```ts
import makeWASocket from '@whiskeysockets/baileys'

const sock = makeWASocket({
    // configurações adicionais aqui
})

if (!sock.authState.creds.registered) {
    const number = '5511999999999' // código do país + DDD + número
    const code = await sock.requestPairingCode(number)
    console.log('Código de pareamento:', code)
}
```

### Recebendo Histórico Completo

1. Defina `syncFullHistory` como `true`.
2. Use a configuração de browser para desktop, que recebe mais histórico de mensagens.

```ts
const sock = makeWASocket({
    ...outrasOpcoes,
    browser: Browsers.macOS('Desktop'), // também funciona com Windows, Ubuntu
    syncFullHistory: true,
})
```

---

## ⚙️ Configurações Importantes do Socket

### Cache de Metadados de Grupo (Recomendado)

Se você usa o Baileys em grupos, configure o `cachedGroupMetadata` para evitar requisições desnecessárias à API:

```ts
import NodeCache from 'node-cache'

const groupCache = new NodeCache({ stdTTL: 5 * 60, useClones: false })

const sock = makeWASocket({
    cachedGroupMetadata: async (jid) => groupCache.get(jid),
})

sock.ev.on('groups.update', async ([event]) => {
    const metadata = await sock.groupMetadata(event.id)
    groupCache.set(event.id, metadata)
})

sock.ev.on('group-participants.update', async (event) => {
    const metadata = await sock.groupMetadata(event.id)
    groupCache.set(event.id, metadata)
})
```

### Melhorar o Sistema de Retry e Descriptografar Votos de Enquete

Para melhorar o reenvio de mensagens em caso de erro e permitir a descriptografia de votos de enquete, você precisa de um store e deve configurar o `getMessage`:

```ts
const sock = makeWASocket({
    getMessage: async (key) => await buscarMensagemDoStore(key),
})
```

### Receber Notificações no App do WhatsApp

Por padrão, quando o Baileys está conectado, o WhatsApp entende que você está online e não envia notificações push para o celular. Para receber notificações, defina:

```ts
const sock = makeWASocket({
    markOnlineOnConnect: false,
})
```

---

## 💾 Salvando e Restaurando Sessão

Para não precisar escanear o QR Code a cada conexão, salve as credenciais com `useMultiFileAuthState`:

```ts
import makeWASocket, { useMultiFileAuthState } from '@whiskeysockets/baileys'

const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')

// Usa as credenciais salvas para conectar — se forem válidas, não precisará de QR Code
const sock = makeWASocket({ auth: state })

// Sempre que as credenciais forem atualizadas, salva automaticamente
sock.ev.on('creds.update', saveCreds)
```

> [!IMPORTANT]
> `useMultiFileAuthState` é uma implementação de referência que salva o estado em uma pasta local. Para produção, recomenda-se implementar o armazenamento em banco de dados (SQL ou NoSQL). As chaves de autenticação (`authState.keys`) são atualizadas a cada mensagem enviada ou recebida — não salvá-las corretamente pode impedir que mensagens cheguem ao destinatário.

---

## 📡 Lidando com Eventos

O Baileys usa a sintaxe de `EventEmitter` para eventos, todos tipados para facilitar o uso com TypeScript.

> [!IMPORTANT]
> Veja todos os eventos disponíveis [aqui](https://baileys.whiskeysockets.io/types/BaileysEventMap.html).

```ts
const sock = makeWASocket()
sock.ev.on('messages.upsert', ({ messages }) => {
    console.log('novas mensagens:', messages)
})
```

### Exemplo Básico

> [!NOTE]
> Este exemplo inclui persistência básica de autenticação e reconexão automática.

```ts
import makeWASocket, { DisconnectReason, useMultiFileAuthState } from '@whiskeysockets/baileys'
import { Boom } from '@hapi/boom'

async function conectarAoWhatsApp() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info_baileys')

    const sock = makeWASocket({ auth: state })

    sock.ev.on('connection.update', (update) => {
        const { connection, lastDisconnect } = update

        if (connection === 'close') {
            const deveReconectar =
                (lastDisconnect?.error as Boom)?.output?.statusCode !== DisconnectReason.loggedOut

            console.log('Conexão encerrada:', lastDisconnect?.error, '| Reconectando:', deveReconectar)

            if (deveReconectar) {
                conectarAoWhatsApp()
            }
        } else if (connection === 'open') {
            console.log('Conexão estabelecida!')
        }
    })

    sock.ev.on('messages.upsert', async (event) => {
        for (const m of event.messages) {
            console.log(JSON.stringify(m, undefined, 2))
            await sock.sendMessage(m.key.remoteJid!, { text: 'Olá!' })
        }
    })

    sock.ev.on('creds.update', saveCreds)
}

conectarAoWhatsApp()
```

> [!IMPORTANT]
> Em `messages.upsert`, use sempre um loop `for...of` sobre `event.messages` para garantir que todas as mensagens do array sejam processadas.

### Descriptografar Votos de Enquete

Votos de enquete chegam criptografados via `messages.update`. Veja como descriptografá-los:

```ts
import { getAggregateVotesInPollMessage } from '@whiskeysockets/baileys'

sock.ev.on('messages.update', async (event) => {
    for (const { key, update } of event) {
        if (update.pollUpdates) {
            const enquete = await buscarMensagemDoStore(key) // implemente no seu lado
            if (enquete) {
                console.log(
                    'Resultado da enquete:',
                    getAggregateVotesInPollMessage({
                        message: enquete,
                        pollUpdates: update.pollUpdates,
                    })
                )
            }
        }
    }
})
```

### Resumo de Eventos na Primeira Conexão

1. `connection.update` é disparado pedindo para reiniciar o socket.
2. O histórico de mensagens é recebido via `messaging.history-set`.

---

## 🗄️ Implementando um Data Store

O Baileys não inclui armazenamento padrão para chats, contatos ou mensagens, mas oferece uma implementação em memória como referência.

> [!IMPORTANT]
> Armazenar o histórico completo de chats em memória é ineficiente. Para produção, implemente seu próprio store com banco de dados.

```ts
import makeWASocket, { makeInMemoryStore } from '@whiskeysockets/baileys'

const store = makeInMemoryStore({})

store.readFromFile('./baileys_store.json')

setInterval(() => {
    store.writeToFile('./baileys_store.json')
}, 10_000)

const sock = makeWASocket({})
store.bind(sock.ev)

sock.ev.on('chats.upsert', () => {
    console.log('Chats:', store.chats.all())
})

sock.ev.on('contacts.upsert', () => {
    console.log('Contatos:', Object.values(store.contacts))
})
```

---

## 🪪 Entendendo os IDs do WhatsApp

O `jid` (também chamado de `id`) é o identificador único de um usuário ou grupo no WhatsApp.

| Tipo | Formato | Exemplo |
|------|---------|---------|
| Usuário | `[código do país][número]@s.whatsapp.net` | `5511999999999@s.whatsapp.net` |
| Grupo | `[id do grupo]@g.us` | `123456789@g.us` |
| Lista de transmissão | `[timestamp]@broadcast` | `1234567890@broadcast` |
| Stories | `status@broadcast` | `status@broadcast` |

> [!NOTE]
> Além do `jid`, o WhatsApp também utiliza o `lid` — um identificador alternativo vinculado à conta do usuário, independente do número de telefone. Ambos são expostos em operações como `onWhatsApp`.

---

## 🛠️ Funções Utilitárias

| Função | Descrição |
|--------|-----------|
| `getContentType(message)` | Retorna o tipo de conteúdo de uma mensagem |
| `getDevice(message)` | Retorna o dispositivo de onde a mensagem foi enviada |
| `makeCacheableSignalKeyStore` | Torna o store de autenticação mais rápido com cache |
| `downloadContentFromMessage` | Faz o download do conteúdo de qualquer mensagem |

---

## 📨 Enviando Mensagens

Todos os tipos de mensagem são enviados com a mesma função:

```ts
const jid: string
const content: AnyMessageContent
const options: MiscMessageGenerationOptions

await sock.sendMessage(jid, content, options)
```

- Veja todos os tipos de conteúdo suportados [aqui](https://baileys.whiskeysockets.io/types/AnyMessageContent.html).
- Veja todas as opções disponíveis [aqui](https://baileys.whiskeysockets.io/types/MiscMessageGenerationOptions.html).

### Mensagens de Texto e Não-Mídia

#### Mensagem de Texto
```ts
await sock.sendMessage(jid, { text: 'Olá, mundo!' })
```

#### Citar uma Mensagem (funciona com todos os tipos)
```ts
await sock.sendMessage(jid, { text: 'Respondendo!' }, { quoted: mensagem })
```

#### Mencionar Usuário (funciona com a maioria dos tipos)
```ts
await sock.sendMessage(jid, {
    text: '@5511999999999 olá!',
    mentions: ['5511999999999@s.whatsapp.net'],
})
```

#### Encaminhar Mensagem
```ts
const msg = buscarMensagemDoStore() // implemente no seu lado
await sock.sendMessage(jid, { forward: msg })
```

#### Mensagem de Localização
```ts
await sock.sendMessage(jid, {
    location: {
        degreesLatitude: -23.5505,
        degreesLongitude: -46.6333,
    },
})
```

#### Mensagem de Contato
```ts
const vcard =
    'BEGIN:VCARD\n' +
    'VERSION:3.0\n' +
    'FN:Nome Completo\n' +
    'TEL;type=CELL;type=VOICE;waid=5511999999999:+55 11 99999-9999\n' +
    'END:VCARD'

await sock.sendMessage(jid, {
    contacts: {
        displayName: 'Meu Contato',
        contacts: [{ vcard }],
    },
})
```

#### Mensagem de Reação
```ts
await sock.sendMessage(jid, {
    react: {
        text: '👍', // string vazia para remover a reação
        key: mensagem.key,
    },
})
```

#### Fixar Mensagem

| Duração | Segundos |
|---------|----------|
| 24 horas | 86.400 |
| 7 dias | 604.800 |
| 30 dias | 2.592.000 |

```ts
await sock.sendMessage(jid, {
    pin: {
        type: 1, // 0 para desafixar
        time: 86400,
        key: mensagem.key,
    },
})
```

#### Enquete
```ts
await sock.sendMessage(jid, {
    poll: {
        name: 'Qual a sua linguagem favorita?',
        values: ['TypeScript', 'Python', 'Go', 'Rust'],
        selectableCount: 1,
        toAnnouncementGroup: false,
    },
})
```

### Mensagens com Prévia de Link

Por padrão, o WhatsApp Web não gera prévias de links. O Baileys suporta isso com a dependência `link-preview-js`:

```bash
yarn add link-preview-js
```

```ts
await sock.sendMessage(jid, {
    text: 'Confira: https://github.com/WhiskeySockets/Baileys',
})
```

### Mensagens de Mídia

> [!NOTE]
> Você pode passar `{ stream: Stream }`, `{ url: Url }` ou diretamente um `Buffer`. Veja mais [aqui](https://baileys.whiskeysockets.io/types/WAMediaUpload.html).

> [!TIP]
> Prefira usar `stream` ou `url` para economizar memória — o Baileys nunca carrega o buffer inteiro na memória ao usar essas opções.

#### GIF
```ts
// WhatsApp não suporta .gif nativamente; use .mp4 com gifPlayback: true
await sock.sendMessage(jid, {
    video: fs.readFileSync('media/animacao.mp4'),
    caption: 'Olha esse gif!',
    gifPlayback: true,
})
```

#### Vídeo
```ts
await sock.sendMessage(jid, {
    video: { url: './media/video.mp4' },
    caption: 'Meu vídeo',
    ptv: false, // true para enviar como "video note" (bolinha)
})
```

#### Áudio

Para garantir compatibilidade em todos os dispositivos, converta o áudio com `ffmpeg`:

```bash
ffmpeg -i entrada.mp4 -avoid_negative_ts make_zero -ac 1 saida.ogg
```

```ts
await sock.sendMessage(jid, {
    audio: { url: './media/audio.ogg' },
    mimetype: 'audio/mp4',
})
```

#### Imagem
```ts
await sock.sendMessage(jid, {
    image: { url: './media/imagem.png' },
    caption: 'Minha imagem',
})
```

#### Mensagem de Visualização Única (View Once)

Funciona com imagem, vídeo e áudio:

```ts
await sock.sendMessage(jid, {
    image: { url: './media/imagem.png' },
    viewOnce: true,
    caption: 'Só você pode ver!',
})
```

---

## ✏️ Modificando Mensagens

### Deletar Mensagem (para todos)
```ts
const msg = await sock.sendMessage(jid, { text: 'ops, errei' })
await sock.sendMessage(jid, { delete: msg.key })
```

> **Nota:** Para deletar apenas para você, use `chatModify` (veja [Modificando Chats](#-modificando-chats)).

### Editar Mensagem
```ts
await sock.sendMessage(jid, {
    text: 'Texto corrigido!',
    edit: mensagemOriginal.key,
})
```

---

## 🖼️ Manipulando Mídia

### Thumbnail em Mensagens de Mídia

Thumbnails são geradas automaticamente para imagens e stickers se você tiver `jimp` ou `sharp` instalado:

```bash
yarn add jimp
# ou
yarn add sharp
```

Para vídeos, é necessário ter o `ffmpeg` instalado no sistema.

### Baixar Mídia Recebida

```ts
import { createWriteStream } from 'fs'
import { downloadMediaMessage, getContentType } from '@whiskeysockets/baileys'

sock.ev.on('messages.upsert', async ({ messages }) => {
    for (const m of messages) {
        if (!m.message) continue

        const tipo = getContentType(m.message)

        if (tipo === 'imageMessage') {
            const stream = await downloadMediaMessage(
                m,
                'stream',
                {},
                {
                    logger,
                    reuploadRequest: sock.updateMediaMessage,
                }
            )
            stream.pipe(createWriteStream('./download.jpeg'))
        }
    }
})
```

### Reenviar Mídia Expirada ao WhatsApp

O WhatsApp remove mídias antigas dos servidores. Para acessá-las, outro dispositivo com a mídia precisa reenviá-la:

```ts
await sock.updateMediaMessage(msg)
```

---

## 📵 Rejeitar Chamada

Os valores de `callId` e `callFrom` são obtidos no evento `call`:

```ts
sock.ev.on('call', async ([call]) => {
    await sock.rejectCall(call.id, call.from)
})
```

---

## 💬 Estados no Chat

### Marcar Mensagens como Lidas

```ts
const key: WAMessageKey
await sock.readMessages([key]) // pode passar múltiplas keys
```

O `messageID` pode ser acessado via `message.key.id`.

### Atualizar Presença

Informa ao destinatário se você está online, digitando, gravando áudio, etc.

```ts
// Opções: 'available', 'unavailable', 'composing', 'recording', 'paused'
await sock.sendPresenceUpdate('composing', jid)
```

> [!NOTE]
> Se um cliente desktop estiver ativo, o WhatsApp não envia notificações push. Para recebê-las, marque o Baileys como offline: `sock.sendPresenceUpdate('unavailable')`.

---

## 🗂️ Modificando Chats

> [!IMPORTANT]
> Modificações incorretas podem resultar no deslogamento de todos os seus dispositivos.

### Arquivar Chat
```ts
const ultimaMensagem = await buscarUltimaMensagem(jid)
await sock.chatModify({ archive: true, lastMessages: [ultimaMensagem] }, jid)
```

### Silenciar / Remover Silêncio

| Duração | Milissegundos |
|---------|--------------|
| Remover | `null` |
| 8 horas | `28.800.000` |
| 7 dias | `604.800.000` |

```ts
await sock.chatModify({ mute: 8 * 60 * 60 * 1000 }, jid) // silenciar por 8h
await sock.chatModify({ mute: null }, jid)               // remover silêncio
```

### Marcar Chat como Lido / Não Lido
```ts
const ultimaMensagem = await buscarUltimaMensagem(jid)
await sock.chatModify({ markRead: false, lastMessages: [ultimaMensagem] }, jid)
```

### Deletar Mensagem para Mim
```ts
await sock.chatModify(
    {
        clear: {
            messages: [{ id: 'ID_DA_MENSAGEM', fromMe: true, timestamp: '1654823909' }],
        },
    },
    jid
)
```

### Deletar Chat
```ts
const ultimaMensagem = await buscarUltimaMensagem(jid)
await sock.chatModify(
    {
        delete: true,
        lastMessages: [{ key: ultimaMensagem.key, messageTimestamp: ultimaMensagem.messageTimestamp }],
    },
    jid
)
```

### Fixar / Desafixar Chat
```ts
await sock.chatModify({ pin: true }, jid)  // fixar
await sock.chatModify({ pin: false }, jid) // desafixar
```

### Favoritar / Desfavoritar Mensagem
```ts
await sock.chatModify(
    {
        star: {
            messages: [{ id: 'ID_DA_MENSAGEM', fromMe: true }],
            star: true, // false para desfavoritar
        },
    },
    jid
)
```

### Mensagens Temporárias (Disappearing)

| Duração | Segundos |
|---------|----------|
| Desativar | `0` |
| 24 horas | `86.400` |
| 7 dias | `604.800` |
| 90 dias | `7.776.000` |

```ts
import { WA_DEFAULT_EPHEMERAL } from '@whiskeysockets/baileys'

// Ativar mensagens temporárias no chat
await sock.sendMessage(jid, { disappearingMessagesInChat: WA_DEFAULT_EPHEMERAL })

// Enviar uma mensagem já como temporária
await sock.sendMessage(jid, { text: 'Texto' }, { ephemeralExpiration: WA_DEFAULT_EPHEMERAL })

// Desativar mensagens temporárias
await sock.sendMessage(jid, { disappearingMessagesInChat: false })
```

---

## 🔍 Consultas de Usuário

### Verificar se um Número Existe no WhatsApp

A função `onWhatsApp` verifica a existência de um número e retorna dois identificadores:

- **`jid`** — identificador principal do WhatsApp, no formato `número@s.whatsapp.net`
- **`lid`** — identificador alternativo vinculado à conta, independente do número de telefone (usado internamente pelo WhatsApp em contextos multi-device e para privacidade de número)

```ts
const [resultado] = await sock.onWhatsApp(jid)

if (resultado.exists) {
    console.log(`Número existe no WhatsApp`)
    console.log(`JID: ${resultado.jid}`)
    console.log(`LID: ${resultado.lid}`)
}
```

### Buscar Histórico de Chat (grupos também)

```ts
const mensagem = await buscarMensagemMaisAntiga(jid) // implemente no seu lado
await sock.fetchMessageHistory(
    50, // quantidade (máx. 50 por consulta)
    mensagem.key,
    mensagem.messageTimestamp
)
// As mensagens serão recebidas no evento 'messaging.history-set'
```

### Buscar Status
```ts
const status = await sock.fetchStatus(jid)
console.log('Status:', status)
```

### Buscar Foto de Perfil (grupos também)
```ts
const urlBaixa = await sock.profilePictureUrl(jid)          // baixa resolução
const urlAlta  = await sock.profilePictureUrl(jid, 'image') // alta resolução
```

### Buscar Perfil Empresarial
```ts
const perfil = await sock.getBusinessProfile(jid)
console.log('Descrição:', perfil.description, '| Categoria:', perfil.category)
```

### Verificar Presença (digitando, online...)
```ts
sock.ev.on('presence.update', console.log)
await sock.presenceSubscribe(jid)
```

---

## 👤 Alterando Perfil

### Alterar Status
```ts
await sock.updateProfileStatus('Disponível!')
```

### Alterar Nome
```ts
await sock.updateProfileName('Meu Nome')
```

### Alterar Foto de Perfil (grupos também)
```ts
await sock.updateProfilePicture(jid, { url: './nova-foto.jpeg' })
```

### Remover Foto de Perfil (grupos também)
```ts
await sock.removeProfilePicture(jid)
```

---

## 👥 Grupos

> [!NOTE]
> Para alterar propriedades de um grupo, você precisa ser **administrador**.

### Criar Grupo
```ts
const grupo = await sock.groupCreate('Meu Grupo', [
    '5511999999999@s.whatsapp.net',
    '5511888888888@s.whatsapp.net',
])
console.log('Grupo criado com ID:', grupo.gid)
```

### Adicionar, Remover, Promover ou Rebaixar
```ts
await sock.groupParticipantsUpdate(
    jid,
    ['5511999999999@s.whatsapp.net'],
    'add' // 'remove' | 'promote' | 'demote'
)
```

### Alterar Nome do Grupo
```ts
await sock.groupUpdateSubject(jid, 'Novo Nome do Grupo')
```

### Alterar Descrição
```ts
await sock.groupUpdateDescription(jid, 'Nova descrição!')
```

### Alterar Configurações
```ts
await sock.groupSettingUpdate(jid, 'announcement')     // apenas admins enviam mensagens
await sock.groupSettingUpdate(jid, 'not_announcement') // todos enviam mensagens
await sock.groupSettingUpdate(jid, 'locked')           // apenas admins alteram configurações
await sock.groupSettingUpdate(jid, 'unlocked')         // todos alteram configurações
```

### Sair do Grupo
```ts
await sock.groupLeave(jid)
```

### Obter Link de Convite
```ts
const codigo = await sock.groupInviteCode(jid)
console.log('Link:', `https://chat.whatsapp.com/${codigo}`)
```

### Revogar Link de Convite
```ts
const novoCodigo = await sock.groupRevokeInvite(jid)
```

### Entrar por Código de Convite
```ts
// Apenas o código, sem a URL completa
const resposta = await sock.groupAcceptInvite(codigo)
```

### Informações do Grupo pelo Código de Convite
```ts
const info = await sock.groupGetInviteInfo(codigo)
```

### Buscar Metadados (participantes, nome, descrição...)
```ts
const metadata = await sock.groupMetadata(jid)
console.log(metadata.id, metadata.subject, metadata.desc)
```

### Entrar por `groupInviteMessage`
```ts
const resposta = await sock.groupAcceptInviteV4(jid, groupInviteMessage)
```

### Listar Solicitações de Entrada
```ts
const solicitacoes = await sock.groupRequestParticipantsList(jid)
```

### Aprovar / Rejeitar Solicitações
```ts
await sock.groupRequestParticipantsUpdate(
    jid,
    ['5511999999999@s.whatsapp.net'],
    'approve' // ou 'reject'
)
```

### Listar Todos os Grupos com Metadados
```ts
const grupos = await sock.groupFetchAllParticipating()
```

### Mensagens Temporárias no Grupo
```ts
await sock.groupToggleEphemeral(jid, 86400) // 0 para desativar
```

### Modo de Adição de Membros
```ts
await sock.groupMemberAddMode(jid, 'all_member_add') // ou 'admin_add'
```

---

## 🔒 Privacidade

### Bloquear / Desbloquear Usuário
```ts
await sock.updateBlockStatus(jid, 'block')   // bloquear
await sock.updateBlockStatus(jid, 'unblock') // desbloquear
```

### Obter Configurações de Privacidade
```ts
const configuracoes = await sock.fetchPrivacySettings(true)
```

### Obter Lista de Bloqueados
```ts
const bloqueados = await sock.fetchBlocklist()
```

### Atualizar Privacidade do "Visto por último"
```ts
await sock.updateLastSeenPrivacy('all') // 'contacts' | 'contact_blacklist' | 'none'
```

### Atualizar Privacidade do Status Online
```ts
await sock.updateOnlinePrivacy('all') // 'match_last_seen'
```

### Atualizar Privacidade da Foto de Perfil
```ts
await sock.updateProfilePicturePrivacy('all') // 'contacts' | 'contact_blacklist' | 'none'
```

### Atualizar Privacidade do Status
```ts
await sock.updateStatusPrivacy('all') // 'contacts' | 'contact_blacklist' | 'none'
```

### Atualizar Privacidade de Confirmação de Leitura
```ts
await sock.updateReadReceiptsPrivacy('all') // 'none'
```

### Atualizar Quem Pode Adicionar a Grupos
```ts
await sock.updateGroupsAddPrivacy('all') // 'contacts' | 'contact_blacklist'
```

### Atualizar Modo Padrão de Mensagens Temporárias
```ts
await sock.updateDefaultDisappearingMode(86400) // em segundos
```

---

## 📢 Listas de Transmissão e Stories

### Enviar para Transmissão ou Story
```ts
await sock.sendMessage(
    jid,
    {
        image: { url: urlDaImagem },
        caption: 'Meu story!',
    },
    {
        backgroundColor: '#000000',
        font: 1,
        statusJidList: ['5511999999999@s.whatsapp.net'], // quem vai receber
        broadcast: true,
    }
)
```

- O corpo pode ser `extendedTextMessage`, `imageMessage`, `videoMessage` ou `voiceMessage`.
- `broadcast: true` ativa o modo de transmissão.
- `statusJidList` define quem receberá o story/transmissão.
- IDs de lista de transmissão têm o formato `timestamp@broadcast`.

### Buscar Informações de uma Lista de Transmissão
```ts
const lista = await sock.getBroadcastListInfo('1234@broadcast')
console.log('Nome:', lista.name, '| Destinatários:', lista.recipients)
```

---

## 🧩 Funcionalidades Customizadas

O Baileys foi projetado para extensibilidade. Em vez de fazer um fork e reescrever os internos, você pode simplesmente escrever suas próprias extensões.

### Ativar Nível de Debug nos Logs

```ts
const sock = makeWASocket({
    logger: P({ level: 'debug' }),
})
```

Isso exibirá no console todas as mensagens recebidas do WhatsApp, útil para entender o protocolo e depurar problemas.

### Como o WhatsApp se Comunica

> [!TIP]
> Para entender o protocolo do WhatsApp, estude o **Libsignal Protocol** e o **Noise Protocol**.

Cada mensagem recebida é um "frame" com três componentes:

- `tag` — o tipo do frame (ex: `message`, `ib`)
- `attrs` — pares chave-valor com metadados (como o ID da mensagem)
- `content` — o conteúdo real da mensagem

Leia mais sobre o formato [aqui](/src/WABinary/readme.md).

### Registrar Callback para Eventos WebSocket

```ts
// Qualquer mensagem com a tag 'edge_routing'
sock.ws.on('CB:edge_routing', (node: BinaryNode) => { })

// Com tag 'edge_routing' e atributo id = abcd
sock.ws.on('CB:edge_routing,id:abcd', (node: BinaryNode) => { })

// Com tag, id e primeiro nó de conteúdo 'routing_info'
sock.ws.on('CB:edge_routing,id:abcd,routing_info', (node: BinaryNode) => { })
```

> [!TIP]
> Veja a função `onMessageReceived` no arquivo `socket.ts` para entender como os eventos WebSocket são disparados internamente.

---

## 🔗 Links Úteis

- [Discord](https://discord.gg/WeJM5FP9GG)
- [Documentação](https://baileys.whiskeysockets.io/)
- [Guia oficial](https://baileys.wiki)
- [Exemplo completo](Example/example.ts)

---

## ⚠️ Aviso Legal

Este projeto não é afiliado, associado, autorizado, endossado ou de qualquer forma oficialmente conectado ao WhatsApp ou a qualquer de suas subsidiárias ou afiliadas.

Os mantenedores do Baileys não conduzem e não condoned o uso desta aplicação em práticas que violem os Termos de Serviço do WhatsApp. Use com responsabilidade. Não faça spam. O uso para stalkerware, envio em massa ou mensagens automatizadas é desencorajado.

---

## 📄 Licença

Copyright (c) 2025 Rajeh Taher / WhiskeySockets — Licenciado sob a [MIT License](LICENSE).
