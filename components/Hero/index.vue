<template>
  <div class="hero">
    <Blob class="blob" />
    <div id="text-container" class="text-container">
      <p
        v-for="(messageItem, index) in messages"
        :id="'message-' + index"
        :key="index"
        class="message-item"
        :style="{
          ...messageItem.styles,
          ...{
            alignSelf:
              messageItem.options.textArea === 'right'
                ? 'flex-end'
                : 'flex-start',
          },
        }"
      >
        {{ messageItem.message }}
      </p>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    scale: {
      type: Number,
      default: 0.6,
    },
  },
  data() {
    return {
      messages: [],
      messagesQueue: [],
      isInitializing: true,
      isBusy: false,
    }
  },

  mounted() {
    this.init()
  },

  methods: {
    init() {
      const self = this
      const blobScale = this.scale

      const onComplete = function () {
        self.$emit('startup-animation-finished')
        self.isInitializing = false
        self._checkMessageQueue()
      }
      this.$KUTE
        .fromTo(
          '.blob',
          { opacity: 0, scale: 0.1, translateX: 0, rotate: 0 },
          { opacity: 1, scale: blobScale, translateX: 250, rotate: 360 },
          { duration: 1500, easing: 'easingCircularInOut', onComplete }
        )
        .start()
    },
    _actiallyMoveBlob(options) {
      const MOVE_BLOB_TWEEN_DURATION = 1500
      const ON_COMPLETE_MOVE_BLOB_DELAY = MOVE_BLOB_TWEEN_DURATION - 800

      const self = this
      const direction = options.direction
      const screenWidth = window.innerWidth
      const blobWidth = document.getElementById('blob').offsetWidth
      // const blobLeftoffset = document
      //   .getElementById('blob')
      //   .getBoundingClientRect().left

      if (direction === 'left') {
        const newValue = (screenWidth / 6) * 1 - blobWidth / 2 + this.blobTransformState.translateX
        const newBlobState = {
          ...this.blobTransformState,
          ...{ translateX: newValue },
        }
        const onComplete = function () {
          self.blobTransformState = newBlobState
          self._checkMessageQueue()
        }
        this.$KUTE
          .fromTo('.blob', this.blobTransformState, newBlobState, {
            duration: MOVE_BLOB_TWEEN_DURATION,
            easing: 'easingSinusoidalInOut',
          })
          .start()

        setTimeout(()=> onComplete(), ON_COMPLETE_MOVE_BLOB_DELAY)
      }
    },
    moveBlob(options) {
      if (
        this.messagesQueue.length === 0 &&
        this.isBusy === false &&
        this.isInitializing === false
      ) {
        this._actiallyMoveBlob(options)
      } else {
        this.messagesQueue.push({message:'>MOVE_BLOB:' + JSON.stringify(options)})
      }
    },
    pushText(message, styles, options) {
      const defaultOptions = {
        textArea: 'left',
      }

      const newOptions = {
        ...defaultOptions,
        ...options,
      }
      if (
        this.messagesQueue.length === 0 &&
        this.isBusy === false &&
        this.isInitializing === false
      ) {
        this.isBusy = true
        this._actuallyPushText({ message, styles, options: newOptions })
      } else {
        this.messagesQueue.push({ message, styles, options: newOptions })
      }
    },
    clearTexts() {
      if (
        this.messagesQueue.length === 0 &&
        this.isBusy === false &&
        this.isInitializing === false
      ) {
        this._actuallyClearTexts()
      } else {
        this.messagesQueue.push({message:'>CLEAR'})
      }
    },
    _actuallyPushText(messageItem) {
      this.messages.push(messageItem)
      this.$nextTick(function () {
        const messageTween = this._getDisplayMessageTween(message)
        messageTween.start()
      })
    },
    _actuallyClearTexts() {
      const BEFORE_CLEAR_DELAY = 1500
      const CLEAR_TEXT_TWEEN_DURATION = 650
      const self = this
      const textContainer = document.getElementsByClassName('message-item')

      const onComplete = function () {
        self.messages = []
        self._checkMessageQueue()
      }

      const clearContainerTween = this.$KUTE.allFromTo(
        textContainer,
        { opacity: 1, translateY: 0 },
        { opacity: 0, translateY: -100 },
        {
          duration: CLEAR_TEXT_TWEEN_DURATION,
          easing: 'easingSinusoidalInOut',
        }
      )
      setTimeout(() => {
        clearContainerTween.start()
        // Need to add another timout so multiple tweens can have only one onComplete() callback
        setTimeout(() => onComplete(), CLEAR_TEXT_TWEEN_DURATION)
      }, BEFORE_CLEAR_DELAY)
    },
    _checkMessageQueue() {
      if (this.messagesQueue.length > 0) {
        const oldestMessage = this.messagesQueue[0].message

        if (oldestMessage === '>CLEAR') {
          this._actuallyClearTexts()
        } else if (oldestMessage.includes('>MOVE_BLOB:')) {
          const options = JSON.parse(oldestMessage.replace('>MOVE_BLOB:', ''))
          this._actiallyMoveBlob(options)
        } else {
          this._actuallyPushText(this.messagesQueue[0])
        }

        this.messagesQueue.splice(0, 1)
      } else {
        this.isBusy = false
      }
    },
    _getDisplayMessageTween() {
      const NEXT_MESSAGE_OFFSET = 800
      const FADE_IN_DURATION = 800
      const FADE_IN_DISTANCE = 100

      const self = this

      const onComplete = function () {
        setTimeout(function () {
          self._checkMessageQueue()
        }, NEXT_MESSAGE_OFFSET)
      }
      const messageIndex = this.messages.length - 1
      const messageElemId = '#message-' + messageIndex

      return this.$KUTE.fromTo(
        messageElemId,
        { opacity: 0, translateY: FADE_IN_DISTANCE },
        { opacity: 1, translateY: 0 },
        {
          duration: FADE_IN_DURATION,
          easing: 'easingSinusoidalInOut',
          onComplete,
        }
      )
    },
  },
}
</script>

<style scoped>
.hero {
  z-index: 10;

  position: relative;

  position: absolute;
  z-index: 99;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.blob {
  z-index: 50px;
}

.text-container {
  position: absolute;
  display: flex;
  flex-direction: column;
  top: 35%;
  left: 15%;
  width: 70%;
  z-index: 999;
  font-size: 2.5rem;
}

.message-item {
  opacity: 0;
  max-width: 50%;
}
</style>
