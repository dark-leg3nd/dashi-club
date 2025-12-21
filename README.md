//by Ander
import fetch from 'node-fetch'
import fs from 'fs'
import FormData from 'form-data'

const DB_PATH = './database.json'
const CATBOX_API = "https://catbox.moe/user/api.php"


function loadDB() {
  try {
    if (fs.existsSync(DB_PATH)) {
      return JSON.parse(fs.readFileSync(DB_PATH, 'utf-8'))
    }
    return {}
  } catch (e) {
    console.error('Error cargando DB:', e)
    return {}
  }
}

function saveDB(data) {
  try {
    fs.writeFileSync(DB_PATH, JSON.stringify(data, null, 2))
  } catch (e) {
    console.error('Error guardando DB:', e)
    throw e
  }
}

function getSocketData(socketJid) {
  const db = loadDB()
  return db[socketJid] || { name: null, banner: null, createdAt: null }
}

function saveSocketData(socketJid, data) {
  const db = loadDB()
  db[socketJid] = {
    ...db[socketJid],
    ...data,
    updatedAt: new Date().toISOString()
  }
  saveDB(db)
}

async function uploadToCatbox(fileBuffer, fileName) {
  const form = new FormData()
  form.append("reqtype", "fileupload")
  form.append("fileToUpload", fileBuffer, fileName)

  const res = await fetch(CATBOX_API, {
    method: "POST",
    body: form
  })

  const url = await res.text()
  if (!url.startsWith("http")) throw new Error("Error al subir a Catbox")

  return url
}

let handler = async (m, { conn, command, usedPrefix, text }) => {
  try {
    const isSocket = [
      conn.user.jid,
      ...global.owner.map(([n]) => `${n}@s.whatsapp.net`)
    ].includes(m.sender)

    if (!isSocket) return m.reply(`❀ El comando *${command}* solo puede ser ejecutado por el Socket.`)

    await m.react('🕓')

    if (command === 'setbanner' || command === 'setbann') {
      const q = m.quoted || m
      const mime = (q.msg || q).mimetype || q.mediaType || ""

      if (!mime) return m.reply(`❀ Envia o responde una imagen para usarla como banner.`)
      if (!mime.startsWith('image/')) return m.reply(`❀ El archivo debe ser una imagen.`)

      let buffer
      try {
        buffer = await q.download?.()
      } catch (e) {
        return m.reply(`❀ Error al descargar la imagen: ${e.message}`)
      }

      if (!buffer) return m.reply(`❀ No pude descargar la imagen.`)

      const ext = mime.split('/')[1] || "jpg"
      const fileName = `banner_${Date.now()}.${ext}`

      const url = await uploadToCatbox(buffer, fileName)

      saveSocketData(conn.user.jid, {
        banner: url,
        createdAt: getSocketData(conn.user.jid).createdAt || new Date().toISOString()
      })

      await m.reply(`❀ Banner actualizado\nꕥ URL: ${url}`)
      await m.react('✅')
    }

    else if (command === 'setname' || command === 'setnam') {
      if (!text) return m.reply(`❀ Uso: ${usedPrefix}${command} <nombre>`)
      saveSocketData(conn.user.jid, {
        name: text,
        createdAt: getSocketData(conn.user.jid).createdAt || new Date().toISOString()
      })

      await m.reply(`❀ Nombre actualizado\nꕥ ${text}`)
      await m.react('✅')
    }

  } catch (e) {
    console.error(e)
    await m.react('✖️')
    m.reply(`❀ Error: ${e.message}`)
  }
}

handler.help = ['setbanner', 'setname']
handler.tags = ['socket']
handler.command = ['setbanner', 'setbann', 'setname', 'setnam']

export default handler

export function getSocketConfig(id) {
  return getSocketData(id)
}
