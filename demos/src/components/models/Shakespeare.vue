<template>
  <div class="demo">
    <transition name="fade">
      <model-status v-if="modelLoading || modelInitializing" 
        :modelLoading="modelLoading"
        :modelLoadingProgress="modelLoadingProgress"
        :modelInitializing="modelInitializing"
        :modelInitProgress="modelInitProgress"
      ></model-status>
    </transition>
    <v-layout row wrap justify-center align-center>
      <v-flex sm12 md6 class="input-container elevation-1">
        <v-text-field
          :disabled="modelLoading || modelInitializing"
          label="input text"
          v-model="inputText"
          multi-line
          rows="10"
          color="primary"
          @input.native="inputChanged"
        ></v-text-field>
        <div>{{output}}</div>
        <div class="input-buttons">
          <v-btn :disabled="modelLoading || modelInitializing" flat bottom right color="primary" @click="clear">
            <v-icon left>clear</v-icon>CLEAR
          </v-btn>
        </div>
      </v-flex>
    </v-layout>
  </div>
</template>

<script>
import _ from 'lodash'
import axios from 'axios'
import ModelStatus from '../common/ModelStatus'

const MODEL_FILEPATH_PROD =
  'https://transcranial.github.io/keras-js-demos-data/imdb_bidirectional_lstm/imdb_bidirectional_lstm.bin'
const MODEL_FILEPATH_DEV = 'demos/data/shakespeare/model.bin'

const ADDITIONAL_DATA_FILEPATHS_DEV = {
  wordIndex: '/demos/data/shakespeare/indices.json',
  wordDict: '/demos/data/shakespeare/words.json',
  testSamples: '/demos/data/imdb_bidirectional_lstm/imdb_dataset_test.json'
}
const ADDITIONAL_DATA_FILEPATHS_PROD = {
  wordIndex:
    'https://transcranial.github.io/keras-js-demos-data/imdb_bidirectional_lstm/imdb_dataset_word_index_top20000.json',
  wordDict:
    'https://transcranial.github.io/keras-js-demos-data/imdb_bidirectional_lstm/imdb_dataset_word_dict_top20000.json',
  testSamples: 'https://transcranial.github.io/keras-js-demos-data/imdb_bidirectional_lstm/imdb_dataset_test.json'
}
const ADDITIONAL_DATA_FILEPATHS =
  process.env.NODE_ENV === 'production' ? ADDITIONAL_DATA_FILEPATHS_PROD : ADDITIONAL_DATA_FILEPATHS_DEV

export default {
  props: ['hasWebGL'],

  components: { ModelStatus },

  created() {
    // store module on component instance as non-reactive object
    this.model = new KerasJS.Model({
      filepath: process.env.NODE_ENV === 'production' ? MODEL_FILEPATH_PROD : MODEL_FILEPATH_DEV,
      gpu: false
    })

    this.model.events.on('loadingProgress', this.handleLoadingProgress)
    this.model.events.on('initProgress', this.handleInitProgress)
  },

  async mounted() {
    await this.model.ready()
    this.loadAdditionalData()
  },

  beforeDestroy() {
    this.model.cleanup()
    this.model.events.removeAllListeners()
  },

  data() {
    return {
      useGPU: false,
      modelLoading: true,
      modelLoadingProgress: 0,
      modelInitializing: true,
      modelInitProgress: 0,
      modelRunning: false,
      input: new Float32Array(1),
      output: '',
      inputText: '',
      inputTextParsed: [],
      wordIndex: {},
      wordDict: {},
      testSamples: [],
    }
  },

  computed: {
    outputColor() {
      const p = this.output[0]
      if (p > 0 && p < 0.5) {
        return `rgba(242, 38, 19, ${1 - p})`
      } else if (p >= 0.5) {
        return `rgba(27, 188, 155, ${p})`
      }
      return '#69707a'
    }
  },

  methods: {
    handleLoadingProgress(progress) {
      this.modelLoadingProgress = Math.round(progress)
      if (progress === 100) {
        this.modelLoading = false
      }
    },
    handleInitProgress(progress) {
      this.modelInitProgress = Math.round(progress)
      if (progress === 100) {
        this.modelInitializing = false
      }
    },
    clear() {
      this.inputText = ''
      this.inputTextParsed = []
      this.output = ''
    },
    loadAdditionalData() {
      this.modelLoading = true
      const reqs = ['wordIndex', 'wordDict', 'testSamples'].map(key => {
        return axios.get(ADDITIONAL_DATA_FILEPATHS[key])
      })
      axios.all(reqs).then(
        axios.spread((wordIndex, wordDict, testSamples) => {
          this.wordIndex = wordIndex.data
          this.wordDict = wordDict.data
          this.testSamples = testSamples.data
          this.modelLoading = false
        })
      )
    },
    max(weights) {
      let max = -1
      let maxIndex = 0
      for (let index = 0; index < weights.length; index++) {
        if (weights[index] < max)
          continue
        maxIndex = index
        max = weights[index]
      }
      return maxIndex
    },
    inputChanged: _.debounce(async function() {
      if (this.modelRunning) return
      if (this.inputText === '') {
        this.inputTextParsed = []
        return
      }

      this.modelRunning = true

      this.inputTextParsed = this.inputText.toLowerCase()

      this.input = new Float32Array(1)
      let outputData
      for (let i = 0; i < this.inputTextParsed.length; i++) {
        this.input[0] = this.wordIndex[this.inputTextParsed.charAt(i)]
        outputData = await this.model.predict({ input: this.input })
      }

      this.output = ''
      for (let i = 0; i < 200; i++) {
        let choice = this.max(outputData.output)
        let char = this.wordDict[choice]
        this.output += char
        this.input[0] = choice
        outputData = await this.model.predict({ input: this.input })
      }
      this.modelRunning = false
    }, 20),
  }
}
</script>

<style scoped lang="postcss">
@import '../../variables.css';

.input-container {
  font-family: var(--font-monospace);
  background-color: whitesmoke;
  padding: 10px 30px;
  margin-bottom: 30px;

  & .input-buttons {
    display: flex;
    align-items: center;
    justify-content: flex-end;
  }
}

.output-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  user-select: none;
  cursor: default;
  margin-bottom: 30px;

  & .output-heading {
    color: var(--color-lightgray);
    font-family: var(--font-monospace);
    font-size: 16px;
    text-align: center;
    margin: 10px;

    & span {
      display: block;
    }
    & span.output-label.positive {
      color: var(--color-green);
    }
    & span.output-label.negative {
      color: var(--color-red);
    }
  }

  & .output-value {
    transition: color 0.3s ease-in-out;
    font-family: var(--font-monospace);
    font-size: 42px;
    margin: 10px;
  }
}

.architecture-container {
  position: relative;
  width: 710px;
  margin: 0 auto;
  display: flex;
  flex-direction: row;
  align-items: center;

  & .bg-line {
    position: absolute;
    top: 50%;
    left: 0;
    height: 5px;
    width: 100%;
    background: whitesmoke;
  }

  & .layer {
    display: inline-block;
    width: 170px;
    margin-right: 10px;
    background: whitesmoke;
    border-radius: 5px;
    padding: 2px 10px 0px;
    z-index: 1;

    & .layer-class-name {
      color: var(--color-green);
      font-size: 14px;
      font-weight: bold;
    }

    & .layer-details {
      color: #999999;
      font-size: 10px;
      font-weight: bold;
    }
  }

  & .layer:last-child {
    margin-right: 0;
  }
}

.lstm-visualization-container {
  min-width: 700px;
  max-width: 900px;
  margin: 20px auto;
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  align-items: center;
  border: 1px solid var(--color-lightgray);
  border-radius: 5px;
  padding: 10px;

  & .lstm-visualization-word {
    font-family: var(--font-monospace);
    font-size: 14px;
    color: var(--color-darkgray);
    padding: 3px 6px;
  }
}

/* vue transition `fade` */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
</style>
