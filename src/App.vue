<template>
  <h1>Hello e2ee !</h1>
  <p>(check the console to see the keys!)</p>
  <textarea v-model="encryptMessage"></textarea>
  <button @click="encrypt(encryptMessage)">Encrypt !</button>
  <br><br>
  <textarea v-model="decryptMessage"></textarea>
  <button @click="decrypt(decryptMessage)">Decrypt !</button>
</template>

<script>
export default {
  name: "App",
  data: function () {
    return {
      privateKey: null,
      publicKey: null,
      encryptMessage: null,
      decryptMessage: null
    }
  },
  mounted: async function () {
    await this.createKeys()

    console.log("privateKey", JSON.parse(this.privateKey))
    console.log("publicKey", JSON.parse(this.publicKey))
  },
  methods: {
    async createKeys() {
      const { privateKey, publicKey } = await window.crypto.subtle.generateKey({
            name: "RSA-OAEP",
            modulusLength: 2048,
            publicExponent: new Uint8Array([ 1, 0, 1 ]),
            hash: { name: "SHA-256" },
          },
          true,
          [ "encrypt", "decrypt" ]
      )

      this.privateKey = await this.exportKeys(privateKey)
      this.publicKey = await this.exportKeys(publicKey)
    },
    async exportKeys(key) {
      const keyData = await window.crypto.subtle.exportKey("jwk", key)
      return JSON.stringify(keyData, null, " ");
    },
    importKey(key, name) {
      const usage = name === "public" ? "encrypt" : "decrypt"
      const jwk = JSON.parse(key)

      return window.crypto.subtle.importKey(
          "jwk",
          jwk,
          {
            name: "RSA-OAEP",
            hash: "SHA-256"
          },
          true,
          [ usage ]
      );
    },
    async encrypt(message) {
      const enc = new TextEncoder();
      const publicKey = await this.importKey(this.publicKey, "public")

      const data = await window.crypto.subtle.encrypt(
          {
            name: "RSA-OAEP",
          },
          publicKey,
          enc.encode(message)
      )

      const encrypted = this._arrayBufferToBase64(data)
      this.encryptMessage = null; this.decryptMessage = encrypted
      return encrypted
    },
    async decrypt(message) {
      const privateKey = await this.importKey(this.privateKey, "private")
      const msg = this._base64ToArrayBuffer(message)

      const data = await window.crypto.subtle.decrypt(
          { name: "RSA-OAEP" },
          privateKey,
          msg
      );

      const enc = new TextDecoder("utf-8");
      this.decryptMessage = enc.decode(data);
    },

    _arrayBufferToBase64(buffer) {
      let binary = ''
      const bytes = new Uint8Array(buffer)
      const len = bytes.byteLength;
      for (let i = 0; i < len; i++) {
        binary += String.fromCharCode(bytes[i]);
      }
      return window.btoa(binary);
    },
    _base64ToArrayBuffer(base64) {
      const binary_string = window.atob(base64);
      const len = binary_string.length;
      const bytes = new Uint8Array(len);
      for (let i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
      }
      return bytes.buffer;
    }
  }
}
</script>
