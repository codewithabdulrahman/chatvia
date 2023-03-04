<script>
import simplebar from "simplebar-vue";

import vClickOutside from "v-click-outside";
import emojis from "vue-emoji-picker/src/emojis";

import Message from "./message";
import { chatMemberList, chatMessages } from "./data";

const { detectMobile } = require("../../utils/mobileDetection");

export default {
  components: {
    simplebar,
    Message
  },
  directives: {
    clickOutside: vClickOutside.directive,
  },
  props: {
    currentUserId: {
      type: [String, Number],
      required: true,
    },
    textMessages: {
      type: Object,
      required: true,
    },
    isMobile: {
      type: Boolean,
      required: true,
    },
    rooms: {
      type: Array,
      required: true,
    },
    id: {
      type: [String, Number],
      required: true,
    },
    messages: {
      type: Array,
      required: true,
    },
    roomMessage: {
      type: String,
    },
    menuActions: {
      type: Array,
      required: true,
    },
    messageActions: {
      type: Array,
      required: true,
    },
    showFiles: {
      type: Boolean,
      required: true,
    },
    showEmojis: {
      type: Boolean,
      required: true,
    },
    showReactionEmojis: {
      type: Boolean,
      required: true,
    },
    textFormatting: {
      type: Boolean,
      required: true,
    },
    loadingRooms: {
      type: Boolean,
      required: true,
    },
    roomInfo: {
      type: Function,
    },
  },
  data() {
    return {
      chatMemberList: chatMemberList,
      chatMessages: chatMessages,
      message: "",
      editedMessage: {},
      messageReply: null,
      infiniteState: null,
      loadingMessages: false,
      loadingMoreMessages: false,
      file: null,
      imageFile: null,
      imageDimensions: null,
      menuOpened: false,
      emojiOpened: false,
      emojisList: {},
      hideOptions: true,
      scrollIcon: false,
      newMessages: [],
    };
  },
  mounted() {
    this.newMessages = [];
    window.addEventListener("keyup", (e) => {
      if (e.keyCode === 13 && !e.shiftKey) {
        if (detectMobile()) {
          this.message = this.message + "\n";
          setTimeout(() => this.onChangeInput(), 0);
        } else {
          this.sendMessage();
        }
      }
    });
    const emojisTable = Object.keys(emojis).map((key) => emojis[key]);
    this.emojisList = Object.assign({}, ...emojisTable);
  },
  watch: {
    loadingMessages(val) {
      if (val) this.infiniteState = null;
      else this.focusTextarea(true);
    },
    room(newVal, oldVal) {
      if (newVal.id && newVal.id !== oldVal.id) {
        this.loadingMessages = true;
        this.scrollIcon = false;
        this.resetMessage(true);
        if (this.roomMessage) {
          this.message = this.roomMessage;
          setTimeout(() => this.onChangeInput(), 0);
        }
      }
    },
    messages(newVal, oldVal) {
      const element = this.$refs.scrollContainer;
      if (!element) return;
      const options = {
        top: element.scrollHeight,
      };
      if (oldVal && newVal && oldVal.length === newVal.length - 1) {
        return setTimeout(() => {
          options.behavior = "smooth";
          element.scrollTo(options);
        }, 50);
      }
      if (this.infiniteState) {
        this.infiniteState.loaded();
        setTimeout(() => (this.loadingMoreMessages = false), 0);
      } else if (newVal.length) {
        setTimeout(() => {
          element.scrollTo(options);
          this.loadingMessages = false;
        }, 0);
      }
    },
    imageFile() {
      setTimeout(() => {
        if (!this.$refs.imageFile) {
          this.imageDimensions = null;
          setTimeout(() => this.resizeTextarea(), 0);
        } else {
          let height = this.$refs.imageFile.height;
          if (height < 30) height = 30;
          this.imageDimensions = {
            height: this.$refs.imageFile.height - 10,
            width: this.$refs.imageFile.width + 26,
          };
        }
      }, 20);
    },
  },
  computed: {
    room() {
      return this.chatMemberList.find((room) => room.id === this.id) || {};
    },
    inputDisabled() {
      return this.isMessageEmpty();
    },
  },
  methods: {
    addNewMessage(message) {
      this.newMessages.push(message);
      this.handleScroll();
    },
    handleScroll() {
      if (this.$refs.current.$el) {
        setTimeout(() => {
          this.$refs.current.SimpleBar.getScrollElement().scrollTop =
            this.$refs.current.SimpleBar.getScrollElement().scrollHeight + 1500;
        }, 500);
      }
    },
    resetMessage(disableMobileFocus = null, editFile = null) {
      this.$emit("typingMessage", null);
      if (editFile) {
        this.file = null;
        this.message = "";
        return;
      }
      this.resetTextareaSize();
      this.message = "";
      this.editedMessage = {};
      this.messageReply = null;
      this.file = null;
      this.imageFile = null;
      this.emojiOpened = false;
      setTimeout(() => this.focusTextarea(disableMobileFocus), 0);
    },
    resetImageFile() {
      this.imageFile = null;
      this.editedMessage.file = null;
      this.file = null;
      this.focusTextarea();
    },
    resetTextareaSize() {
      if (!this.$refs["roomTextarea"]) return;
      this.$refs["roomTextarea"].style.height = "20px";
    },
    focusTextarea(disableMobileFocus) {
      if (detectMobile() && disableMobileFocus) return;
      this.$refs["roomTextarea"].focus();
    },
    isMessageEmpty() {
      return !this.file && !this.message.trim();
    },
    sendMessage() {
      if (!this.file && !this.message.trim()) return;
      if (this.editedMessage.id) {
        if (this.editedMessage.content !== this.message || this.file) {
          this.$emit("editMessage", {
            messageId: this.editedMessage.id,
            newContent: this.message.trim(),
            file: this.file,
            replyMessage: this.messageReply,
          });
        }
      } else {
        const currentDate = new Date();
        this.$emit("sendMessage", {
          id: this.chatMessages.length + 1,
          content: this.message.trim(),
          file: this.file,
          replyMessage: this.messageReply,
          time: currentDate.getHours() + ":" + currentDate.getMinutes(),
          align: "right",
          image: require("@/assets/images/users/avatar-1.jpg"),
          name: "Patricia Smith",
        });
      }
      this.resetMessage(true);
    },
    messageActionHandler({ action, message }) {
      switch (action.name) {
        case "Reply":
          return this.replyMessage(message);
        case "Edit":
          return this.editMessage(message);
        case "Delete":
          return this.$emit("deleteMessage", message.id);
        default:
          return this.$emit("messageActionHandler", {
            action,
            message,
          });
      }
    },
    sendMessageReaction(messageReaction) {
      this.$emit("sendMessageReaction", messageReaction);
    },
    /**
     * Reply message
     */
    replyMessage(message) {
      this.resetMessage();
      this.messageReply = message;
    },
    /**
     * Edit message
     */
    editMessage(message) {
      this.resetMessage();
      this.editedMessage = {
        ...message,
      };
      this.file = message.file;
      if (this.isImageCheck(this.file)) this.imageFile = message.file.url;
      this.message = message.content;
      setTimeout(() => this.resizeTextarea(), 0);
    },
    scrollToBottom() {
      const element = this.$refs.scrollContainer;
      element.scrollTo({
        top: element.scrollHeight,
        behavior: "smooth",
      });
    },
    onChangeInput() {
      this.resizeTextarea();
      this.$emit("typingMessage", this.message);
    },
    /**
     * Resize textarea
     */
    resizeTextarea() {
      const el = this.$refs["roomTextarea"];
      if (!el) return;
      const padding = window
        .getComputedStyle(el, null)
        .getPropertyValue("padding-top")
        .replace("px", "");
      el.style.height = 0;
      el.style.height = el.scrollHeight - padding * 2 + "px";
    },
    /**
     * Add emoji
     */
    addEmoji(emoji) {
      this.message += emoji.icon;
      this.focusTextarea(true);
    },
    launchFilePicker() {
      this.$refs.file.value = "";
      this.$refs.file.click();
    },
    /**
     * On file changes
     */
    async onFileChange(files) {
      this.resetImageFile();
      const file = files[0];
      const fileURL = URL.createObjectURL(file);
      const blobFile = await fetch(fileURL).then((res) => res.blob());
      this.file = {
        blob: blobFile,
        name: file.name.split(".")[0],
        size: file.size,
        type: file.name.split(".")[1] || file.type,
        localUrl: fileURL,
      };
      if (this.isImageCheck(this.file)) this.imageFile = fileURL;
      else this.message = file.name;
    },
    isImageCheck(file) {
      if (!file) return;
      const imageTypes = ["png", "jpg", "jpeg", "svg"];
      const { type } = file;
      return imageTypes.some((t) => type.includes(t));
    },
    openFile(message) {
      this.$emit("openFile", message);
    },
    menuActionHandler(action) {
      this.closeMenu();
      this.$emit("menuActionHandler", action);
    },
    closeMenu() {
      this.menuOpened = false;
    },
    toggleProfile() {
      document.getElementById("profile-show").style.display = "block";
    },
    closeUserChat() {
      var userChat = document.getElementsByClassName("user-chat");
      if (userChat) {
        userChat[0].classList.remove("user-chat-show");
      }
    },
    hideProfile() {
      document.getElementById("profile-show").style.display = "none";
    },
  },
};
</script>

<template>
  <div class="ai-chat-row full-width ">
    <div class="wrapper">
      <div class="user-chat w-100 overflow-hidden">
        <div class="d-lg-flex">
          <div class="w-100 overflow-hidden position-relative">
            <simplebar class="chat-conversation p-3 p-lg-4" id="containerElement" ref="current">
              <div v-for="(message, index) in messages" :key="index">
                <message :currentUserId="currentUserId" :message="message" :messages="chatMessages"
                  :editedMessage="editedMessage" :messageActions="messageActions" :roomUsers="room.users"
                  :textMessages="textMessages" :roomFooterRef="$refs.roomFooter" :newMessages="newMessages"
                  :showReactionEmojis="showReactionEmojis" :textFormatting="textFormatting" :emojisList="emojisList"
                  :hideOptions="hideOptions" @messageActionHandler="messageActionHandler" @openFile="openFile"
                  @addNewMessage="addNewMessage" @sendMessageReaction="sendMessageReaction"
                  @hideOptions="hideOptions = $event"></message>
              </div>
            </simplebar>

            <div ref="roomFooter" class="room-footer">

              <div class="box-footer chat-input-section p-3 p-lg-4 mb-0">
                <div class="image-container" v-if="imageFile">
                  <div class="svg-button icon-image" @click="resetImageFile">
                    <i class="ri-close-circle-fill"></i>
                  </div>
                  <div class="image-file">
                    <img ref="imageFile" :src="imageFile" />
                  </div>
                </div>

                <div v-else-if="file" class="file-container bg-light"
                  :class="{ 'file-container-edit': editedMessage._id }">
                  <div class="icon-file">
                    <i class="ri-file-download-line"></i>
                  </div>
                  <div class="file-message">{{ message }}</div>
                  <div class="svg-button icon-remove" @click="resetMessage(null, true)">
                    <i class="ri-close-line"></i>
                  </div>
                </div>

                <textarea v-show="!file || imageFile" ref="roomTextarea" :placeholder="textMessages.TYPE_MESSAGE" :style="{
                  'min-height': `${imageDimensions ? imageDimensions.height : 20
                    }px`,
                  'padding-left': `${imageDimensions ? imageDimensions.width - 10 : 12
                    }px`,
                }" class="form-control form-control-lg bg-light border-light rounded" v-model="message"
                  @input="onChangeInput" @keydown.esc="resetMessage" @keydown.enter.exact.prevent></textarea>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="footer">
      <div class="col-md-12 copyright-container">
        <img src="../../../assets/images/customgpt-brain2.png" alt="">
        <p>Powered by CustomGPT</p>
      </div>
    </div>


  </div>
</template>

<style lang="scss" scoped>
.full-width {
  background: #fff;
  color: #000;
  // background-image: url('path/to/search-icon.png');
  background-position: left center;
  background-repeat: no-repeat;
  background: url('../../../assets/images/background.png');
  background-position: center;
  background-repeat: no-repeat;
  background-size: cover;
  height: 100vh;
  background-color: #f2f2f2;
}


.copyright-container {
  display: flex;
  align-items: center;
  justify-content: center;
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  padding: 10px;
  box-sizing: border-box;
}

.copyright-container p {
  margin-bottom: 0px;
  margin-left: 3px;
  font-style: normal;
  font-weight: 500;
  font-size: 13px;
  line-height: 27px;
  text-align: center;
  color: #6B6B6B;
}
</style>
