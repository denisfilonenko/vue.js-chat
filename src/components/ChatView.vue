<template>
  <div id="chat_container">
    <div id="chat_box" ref="box">
      <div v-if="new_message" class="chat_message new_message" @click="scrollToEnd">
        у Вас нове повідомлення
      </div>
      <v-btn
          class="btn_bottom mx-2"
          v-if="box_offset - scroll > box_height "
          fab
          dark
          large
          color="#000"
          width="50px"
          height="50px"
          @click="scrollToEnd"
      >
        <svg width="45px" height="45px" viewBox="0 0 24 24">
          <path fill="currentColor" d="M7.41,8.58L12,13.17L16.59,8.58L18,10L12,16L6,10L7.41,8.58Z"/>
        </svg>
      </v-btn>
      <div
          v-for="(message, index) in messages"
          :key="message.id"
          :id="'message' + (index + 1)"
          class="chat_message"
          @click="admin && guid === message.user.guid && editMessage(message, index)"
          :class="guid === message.user.guid ? 'my_message' : 'admin_message'"
          :style="{marginTop: message.another_date ? '35px' : '0px', minHeight: old_device && ((message.files.length && message.files[0].mime.indexOf('image') !== -1) || (message.files.length && message.files[0].mime.indexOf('video') !== -1)) ? '200px' : !old_device ? 'none' : '30px'}"
      >
        <p v-if="message.another_date" class="date">{{ message.date | date_filter(message.another_date) }}</p>
        <textarea v-if="message.isEditing"
                  type="text" v-model="message.message"
                  @blur="message.isEditing = false; editing = false; $emit('update')"
                  v-focus
                  cols="30"
        ></textarea>
        <p v-if="!message.isEditing">{{ message.message }}</p>
        <img v-if="message.files.length && message.files[0].mime.indexOf('image') !== -1"
             :src="image_url + message.files[0].name"
             :alt="message.files[0].name"
             @click="openImage(message.files[0])"
        >
        <div class="video_box"
             v-if="message.files.length && message.files[0].mime.indexOf('video') !== -1"
             @click="openImage(message.files[0])"
        >
          <video preload="metadata">
            <source :src="image_url + message.files[0].name + '#t=0.1'" :type="message.files[0].mime">
          </video>
          <svg
              class="video_play"
              viewBox="0 0 24 24">
            <path fill="currentColor"
                  d="M10,16.5V7.5L16,12M12,2A10,10 0 0,0 2,12A10,10 0 0,0 12,22A10,10 0 0,0 22,12A10,10 0 0,0 12,2Z"/>
          </svg>
        </div>
        <div class="time">
          {{ message.time }}
        </div>
      </div>
      <div v-if="chat_closed" class="chat_message chat_closed">
        Чат закритий адміністратором
      </div>
    </div>
    <v-form
        ref="form"
        @submit.prevent="createMessage"
        v-if="!chat_closed"
    >
      <v-file-input
          v-model="file"
          hide-input
          truncate-length="15"
          class="file_input"
          ref="file"
          @change="handleFile"
      ></v-file-input>
      <v-textarea
          ref="form_area"
          rows="1"
          auto-grow
          class="form_input"
          color="#000 !important"
          v-model="text"
          label="Повідомлення..."
          filled
          background-color="#fff"
          shaped
          :rules="rules"
          @input="typing"
          @blur="resetValidation"
          hide-details
      >
        <v-icon slot="prepend-inner" color="#000" @click="uploadFile">
          {{ file_icon }}
        </v-icon>
        <v-icon slot="append" color="#000" @click="createMessage">
          {{ icon }}
        </v-icon>
      </v-textarea>
      <div v-if="file" class="file_counter">1</div>
    </v-form>
    <v-dialog
        class="image_dialog"
        v-model="dialog"
        fullscreen
        transition="dialog-bottom-transition"
        overlay-color="#000"
        max-width="100%"
    >
      <div class="dialog_box">
        <div class="image_head">
          <a v-if="full_image" class="download_image" :href="image_url + full_image.name" :download="full_image.name"
             target="_blank">
            <svg style="width:30px;height:30px" viewBox="0 0 24 24">
              <path fill="currentColor" d="M5,20H19V18H5M19,9H15V3H9V9H5L12,16L19,9Z"/>
            </svg>
          </a>
          <!--            <v-btn v-if="progress === 0" icon width="45px" height="45px" dark @click="downloadImage">-->
          <!--              <svg style="width:30px;height:30px" viewBox="0 0 24 24">-->
          <!--                <path fill="currentColor" d="M5,20H19V18H5M19,9H15V3H9V9H5L12,16L19,9Z"/>-->
          <!--              </svg>-->
          <!--            </v-btn>-->
          <v-progress-circular
              v-if="progress > 0"
              :rotate="360"
              :size="24"
              :width="6"
              :value="progress"
              color="#fff"
              style="margin-right: 10px;"
          >
          </v-progress-circular>
          <v-btn icon width="45px" height="45px" dark @click="closeImage">
            <svg style="width:34px;height:34px" viewBox="0 0 24 24">
              <path fill="currentColor"
                    d="M19,6.41L17.59,5L12,10.59L6.41,5L5,6.41L10.59,12L5,17.59L6.41,19L12,13.41L17.59,19L19,17.59L13.41,12L19,6.41Z"/>
            </svg>
          </v-btn>
        </div>
        <img v-if="full_image && full_image.mime.indexOf('image') !== -1" class="full_image"
             :src="image_url + full_image.name" :alt="full_image.name">
        <video class="full_image"
               controls="controls"
               v-if="full_image && full_image.mime.indexOf('video') !== -1"
               preload="metadata"
        >
          <source :src="image_url + full_image.name + '#t=0.1'" :type="full_image.mime">
        </video>
      </div>
    </v-dialog>
  </div>
</template>

<script>
import {mdiSendCircleOutline, mdiPaperclip} from '@mdi/js';


export default {
  name: "Chat",
  data: () => ({
    show_chat: false,
    open_answer: false,
    io: null,
    icon: mdiSendCircleOutline,
    file_icon: mdiPaperclip,
    chat: null,
    token: null,
    messages: [],
    text: "",
    rules: [v => !!v || "Text is required"],
    guid: null,
    file: null,
    image_url: 'https://storage.taxiua.app/',
    dialog: false,
    full_image: null,
    progress: 0,
    scroll: 0,
    box_offset: 0,
    box_height: 0,
    questions: null,
    firebase: null,
    old_device: false,
    chat_closed: false,
    new_message: false,
    admin: false,
    editing: false,
    editingMessage: null
  }),
  created() {
    this.messages = []
    this.getToken()
    this.setForOldDevice()

    let self = this

    window.addEventListener('message', event => {
      if (event.data.type === 'input_message_text') {
        self.text = event.data.data;
      }
    });
  },
  updated() {
    let self = this;
    this.$nextTick().then(function () {
      let el = self.$refs.box;
      el.addEventListener('scroll', self.handleScroll)

      document.addEventListener('keypress', function (ev) {
        if (!ev.ctrlKey && !ev.shiftKey && (ev.keyCode === 10 || ev.keyCode === 13)) {
          ev.preventDefault()
          if (self.editing) {
            self.sendEditedMessage()
            return
          }
          if (self.$refs.form_area.isFocused) {
            self.createMessage()
          }

        }
      })

      if (el.scrollTop === 0 && self.box_offset === 0) {
        self.scrollToEnd()
      }

      if (self.new_message && !(self.box_offset - self.scroll > self.box_height)) {
        self.scrollToEnd()
      }
    })
  },
  filters: {
    date_filter(date, month) {
      if (!date) {
        return null
      }
      let new_date = new Date(date)

      if (month) {
        let new_month = new_date.getMonth() + 1;
        if (new_month < 10) {
          new_month = '0' + new_month;
        }
        return new_date.getDate() + '.' + new_month + '.' + new_date.getFullYear()
      }

      let minutes = new_date.getMinutes()
      if (minutes < 10) {
        minutes = '0' + new_date.getMinutes()
      }
      return new_date.getHours() + ':' + minutes
    }
  },
  methods: {
    sendEditedMessage() {
      let message = this.messages[this.editingMessage]
      let payload = {
        message_id: message.id,
        message: message.message
      }

      this.io.emit('message.change', payload)
      this.messages[this.editingMessage].isEditing = false;
      this.editingMessage = null;
      this.editing = false;
    },
    editMessage(message, index) {
      this.editing = true;
      this.editingMessage = index;
      this.messages[index].isEditing = true;
    },
    setDate: function (new_message) {
      let date = new Date(this.messages[0].date)
      let minutes = date.getMinutes()
      if (minutes < 10) {
        minutes = '0' + minutes
      }
      this.messages[0].time = date.getHours() + ':' + minutes;
      this.messages[0].another_date = true
      if (this.messages.length === 1) {
        return;
      }
      if (new_message) {
        let date = new Date(new_message.date)
        let prev_dare = new Date(this.messages[this.messages.length - 2].date)
        if (date.getHours() === prev_dare.getHours() && date.getMinutes() === prev_dare.getMinutes()) {
          this.messages[this.messages.length - 1].time = null
          return
        }
        let new_minutes = date.getMinutes()
        if (new_minutes < 10) {
          new_minutes = '0' + new_minutes
        }
        this.messages[this.messages.length - 1].time = date.getHours() + ':' + new_minutes
        return
      }

      for (let i = 1; i < this.messages.length - 1; i++) {
        let new_date = new Date(this.messages[i].date)
        let old_date = new Date(this.messages[i - 1].date)
        if (new_date.getHours() === old_date.getHours() && new_date.getMinutes() === old_date.getMinutes()) {
          this.messages[i].time = null
        } else {
          let new_minutes = new_date.getMinutes()
          if (new_minutes < 10) {
            new_minutes = '0' + new_minutes
          }
          this.messages[i].time = new_date.getHours() + ':' + new_minutes;
        }

        if (new_date.getDate() !== old_date.getDate() && new_date.getMonth() !== old_date.getMonth() && new_date.getFullYear() !== old_date.getFullYear()) {
          this.messages[i].another_date = true
        }
      }
    },
    setForOldDevice() {
      var version = navigator.userAgent.split('Chrome/')[1].split(' ')[0].split('.')[0]
      if (+version <= 40) {
        this.old_device = true
      }
    },
    getToken: async function () {
      const urlParams = new URLSearchParams(window.location.search)

      this.firebase = urlParams.get('firebase') || null

      if (urlParams.get('admin')) {
        this.admin = true
      }

      if (localStorage.getItem('tu_chat_token') && !urlParams.get('admin')) {
        this.token = localStorage.getItem('tu_chat_token')
      } else {
        if (!urlParams.get('token')) {
          await this.axios.get('https://admin.taxiua.app/api/v2/chats/auth?firebase=' + this.firebase)
              .then(res => {
                this.token = res.data.token
                localStorage.setItem('tu_chat_token', res.data.token)
              })
              .catch(error => {
                console.log(error);
              })
        } else {
          this.token = urlParams.get('token')
          localStorage.setItem('tu_chat_token', this.token)
        }
      }

      if (!urlParams.get('chat_id')) {
        await this.createChat()
      } else {
        this.chat = urlParams.get('chat_id')
      }

      await this.getGuid(this.token)
      this.connectSocket()

    },
    handleScroll() {
      let el = this.$refs.box;
      this.scroll = el.scrollTop;
    },
    createChat: async function () {
      let info = {
        name: 'Не може ввійти',
        type: 'support'
      }

      await this.axios.post('https://chats.taxiua.app/api/users/chats/create?token=' + this.token, info)
          .then(res => {
            this.chat = res.data.id
          })
          .catch(error => {
            console.log(error);
          })
    },
    // listenKeyboard() {
    //   let self = this
    //   let bot_nav = document.getElementById('navigation_bar')
    //   let container = document.getElementById('chat_container')
    //   window.addEventListener('keyboardDidShow', function () {
    //     bot_nav.style.display = 'none'
    //     container.style.height = '100%'
    //   });
    //   window.addEventListener('keyboardDidHide', function () {
    //     bot_nav.style.display = 'flex'
    //     container.style.height = 'calc(100% - 75px)'
    //   });
    //   document.addEventListener('input', function () {
    //     self.changed = true
    //   })
    // },
    getGuid(token) {
      this.guid = JSON.parse(atob(token.split('.')[1])).guid
      // this.getHistory()
    },
    openImage(image) {
      this.full_image = image;
      this.dialog = true
    },
    closeImage() {
      this.full_image = null
      this.dialog = false
      this.progress = 0
    },
    downloadImage() {
      let self = this
      let pdfPath = this.image_url + this.full_image.name

      let permissions = window.cordova.plugins.permissions
      permissions.checkPermission(permissions.WRITE_EXTERNAL_STORAGE, checkPermissionCallback, null)

      function checkPermissionCallback(status) {
        if (!status.hasPermission) {
          var errorCallback = function () {
            console.log('Storage permission is not turned on')
          }
          permissions.requestPermission(
              permissions.WRITE_EXTERNAL_STORAGE,
              function (status) {
                if (!status.hasPermission) {
                  errorCallback()
                } else {
                  downloadFile()
                }
              },
              errorCallback)
        } else {
          downloadFile()
        }
      }

      function downloadFile() {
        let filePath = window.cordova.file.externalRootDirectory + 'Download/' + self.full_image.name
        let fileTransfer = new window.FileTransfer()
        let uri = encodeURI(decodeURIComponent(pdfPath))

        fileTransfer.onprogress = function (ev) {
          self.progress = (ev.lengthComputable) ? Math.round(ev.loaded / ev.total * 100) : 0;
        }

        fileTransfer.download(uri, filePath,
            function () {
              window.cordova.plugins.fileOpener2.open(
                  filePath,
                  self.full_image.mime,
                  {
                    error: function (err) {
                      console.log(err)
                    },
                    success: function () {
                    }
                  }
              )
            },
            function (error) {
              console.log('error')
              console.log(error)
            },
            true,
        )
      }
    },
    uploadFile() {
      this.$refs.file.$refs.input.click()
    },
    handleFile() {
      this.file = this.$refs.file.$refs.input.files[0]
    },
    scrollToEnd: function () {
      let el = this.$refs.box;
      this.box_height = el.clientHeight;
      let scroll_el = document.getElementById('message' + this.messages.length);

      if (scroll_el) {
        let scroll = scroll_el?.offsetTop
        el.scrollTop = scroll
        this.box_offset = scroll
        setTimeout(function () {
          el.scrollTop = scroll
        }, 200)
      }
      this.new_message = false

    },
    // getHistory() {
    //   this.axios.get('https://chats.taxiua.app/api/users/chats?token=' + this.token)
    //       .then(res => {
    //         if (res?.data[0]?.status === 'open') {
    //           this.chat = res.data[0];
    //           this.connectSocket();
    //           this.show_chat = true;
    //           return
    //         }
    //       })
    //       .catch(error => {
    //         console.log(error);
    //       })
    // },
    connectSocket() {
      this.io = require("socket.io-client")("https://chats.taxiua.app", {
        query: {token: this.token, chat_id: this.chat}, transports: ["websocket"]
      });
      const {io} = this;
      const self = this;

      io.on('connect', function () {
        self.messages = [];
      })
      io.on('sync.messages', function (data) {
        for (let i = data.length - 1; i >= 0; i--) {
          let dat = data[i];
          dat.isEditing = false;
          self.messages.unshift(dat)
        }
        self.setDate()
      })
      io.on('message.new', function (data) {
        let dat = data;
        dat.isEditing = false;
        self.messages.push(dat)
        self.new_message = true
        if (self.guid !== data.user.guid) {
          const audio = new Audio("https://storage.taxiua.app/chat_2_notification.wav");
          audio.play();
        }
        self.setDate(data)
      })

      io.on('message.update', async (data) => {
        for (let message of self.messages) {
          if (message.id === data.id) {
            message.message = data.message;
          }
        }
      });

      io.on("kick", function () {
        self.chat_closed = true
      })

      // this.listenKeyboard()
    },
    createMessage() {
      if (this.file) {
        let data = new FormData();
        data.append("files", this.file, this.file.name);
        data.append("chat", this.chat);
        data.append("message", this.text);

        this.file = null
        this.text = ''
        this.resetValidation();

        this.axios.post('https://chats.taxiua.app/api/users/messages/send?token=' + this.token, data)
            .then(() => {
              this.scrollToEnd()
            })
            .catch(error => {
              console.log(error);
            })
        return
      }
      if (this.$refs.form.validate()) {
        this.io.emit("message", {
          message: this.text,
          type: 'text'
        })
        this.scrollToEnd()
        this.text = '';
        this.resetValidation();
      }
    },
    resetValidation() {
      this.$refs.form.resetValidation();
    },
    typing() {

    },
  },
  directives: {
    focus: {
      inserted(el) {
        el.focus()
      }
    }
  },
  beforeDestroy() {
    if (this.show_chat) {
      this.io.disconnect();
    }
  }
};
</script>

<style scoped>
#chat_container {
  background-color: #FAF9FE;
  font-family: 'Red Hat Display', sans-serif;
  position: relative;
  overflow: hidden;
}

#chat_box {
  height: 100vh;
  overflow: auto;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding-top: 10px;
}

#chat_box::-webkit-scrollbar {
  width: 5px;
  color: #212121;
}

#chat_box::-webkit-scrollbar-thumb {
  background-color: #212121;
}

form {
  position: absolute;
  bottom: 0px;
  left: 0px;
  right: 0px;
}

.form_input {
  border-radius: 0;
  z-index: 202;
}

.chat_message:last-child {
  margin-bottom: 75px !important;
}

.chat_message {
  max-width: 80%;
  padding: 5px 10px;
  margin: 3px 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  border-radius: 15px;
  position: relative;
}

.chat_message textarea {
  padding: 0 5px;
  line-height: 1.15;
}
.chat_message textarea:focus {
  outline: none;
}

.chat_message img, .chat_message video {
  display: block;
  max-width: 300px;
  border-radius: 10px;
  margin-bottom: 5px;
  margin-top: 5px;
  cursor: pointer;
}

.chat_message p {
  margin: 0;
  word-break: break-word;
  white-space: pre-wrap;
  line-height: 1.25;
}

.time {
  position: absolute;
  font-size: 10px;
}

.date {
  position: absolute;
  word-break: normal !important;
  top: -30px;
  background-color: rgba(0, 0, 0, .4);
  border-radius: 10px;
  padding: 3px 10px;
  padding-bottom: 0;
  color: white;
}

.admin_message .date {
  left: calc(50vw - 50px);
}

.my_message .date {
  right: calc(50vw - 50px);
}

.my_message {
  background-color: rgba(172, 243, 175, .4);
  text-align: right;
  align-self: flex-end;
}

.my_message .time {
  left: -40px;
  top: 3px;
  align-self: flex-end;
}

.admin_message {
  text-align: left;
  background-color: #212121;
  color: #fff;
}

.admin_message .time {
  align-self: flex-start;
  right: -40px;
  top: 3px;
  color: #212121;
}

.file_input {
  visibility: hidden;
}

.dialog_box {
  position: fixed;
  left: 0;
  right: 0;
  bottom: 0;
  top: 44px;
  background-color: #000;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.image_dialog {
  z-index: 9999 !important;
}

.image_head {
  height: 45px;
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  background-color: #212121;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding: 0 10px;
}

.full_image {
  width: auto;
  height: auto;
  max-width: 100%;
  max-height: 100%;
  margin: 0 auto;
}

.file_counter {
  position: absolute;
  right: 10px;
  bottom: 12px;
  width: 14px;
  height: 14px;
  font-size: 12px;
  color: #fff;
  font-weight: bold;
  border-radius: 50%;
  background-color: #ff0000;
  line-height: 1.3;
  z-index: 999;
  text-align: center;
}

.video_box {
  position: relative;
}

.video_play {
  position: absolute;
  left: calc(50% - 20px);
  bottom: calc(50% - 20px);
  width: 40px;
  height: 40px;
  cursor: pointer;
}

.video_play path {
  fill: #fff;
}

.btn_bottom {
  position: absolute;
  right: 0;
  bottom: 65px;
  z-index: 99;
}

.new_message {
  z-index: 2;
  background-color: rgba(0, 0, 0, .4);
  color: #fff;
  position: absolute;
  bottom: 65px;
  left: 5%;
}

::v-deep .v-textarea textarea {
  line-height: 1.25em;
}

#item_group {
  height: 100%;
  margin: 0 auto;
  display: block;
}

.item {
  height: auto !important;
  margin: 0 auto;
  width: 94%;
  max-width: 640px;
  position: relative;
  margin-bottom: 6px;
}

.item:hover {
  background-color: transparent !important;
}

::v-deep .v-expansion-panel-header {
  box-shadow: 0px 5px 5px rgba(0, 0, 0, 0.03);
  border-radius: 8px !important;
  padding: 25px 14px;
  background-color: #ffffff;
}

::v-deep .v-expansion-panel-content {
  width: 94%;
  margin: 0 auto;
  padding-top: 10px;
  overflow: hidden;
  font-size: 12px;
}

.v-expansion-panel::before {
  box-shadow: none;
}

::v-deep .v-expansion-panel-content ol {
  padding-left: 10px;
}

::v-deep .v-expansion-panel-content li {
  max-width: 300px;
  margin-bottom: 20px;
}

::v-deep .theme--light.v-expansion-panels .v-expansion-panel:not(:first-child):after {
  border: none;
}

h1 {
  font-size: 18px;
  margin: 10px;
  text-align: center;
}

h2 {
  font-size: 16px;
  width: 95%;
}

.item_header {
  font-weight: bold;
  font-size: 16px;
}

.item_content {
  font-size: 16px;
  padding-bottom: 10px;
}

.item_content p {
  margin-bottom: 0;
}

::v-deep .v-expansion-panel-content__wrap {
  padding-left: 0;
  padding-right: 0;
}

.item_content > div > div {
  display: flex;
  justify-content: space-evenly;
  margin-top: 10px;
}

.item_button {
  width: 100px !important;
}

.answer_modal {
  padding: 15px;
  text-align: center;
  font-size: 16px;
}

.download_image {
  color: #fff;
  width: 34px;
  height: 34px;
  margin-top: 2px;
}

.chat_closed {
  background-color: #d84315;
  color: #fff;
  font-weight: bold;
  align-self: center;
  margin-top: 20px;
}

</style>