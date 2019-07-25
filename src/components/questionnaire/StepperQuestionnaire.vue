<template>
  <!-- show every question on a different page (see https://vuetifyjs.com/en/components/steppers) -->
  <div class="card">
    <div class="column card-body">
      <!-- SPINNER -->
      <div v-if="spinner.loading" class="center-vertical">
        <spinner size="large" class="mt4" :message="spinner.message"></spinner>
      </div>
      <!-- PROGRESS -->
      <div v-if="!spinner.loading">
        <div class="progress">
          <div
            class="progress-bar"
            role="progressbar"
            :aria-valuenow="questionCount"
            aria-valuemin="1"
            :aria-valuemax="numberOfQuestions"
            :style="{
              width: (questionCount / numberOfQuestions) * 100 + '%'
            }"
          ></div>
        </div>
        <!-- Progress Counter -->
        <div class="progress-counter title">
          <span :style="{ color: this.primary }" v-if="language"> {{ language.question }} {{ questionCount }}</span
          >&nbsp;
          <span class="color-grey" v-if="language">{{ language.of }} {{ numberOfQuestions }}</span>
        </div>
      </div>
      <br />
      <div v-if="!spinner.loading">
        <component
          :is="getQuestionType"
          :question="getQuestion"
          mode="STEPPER"
          :questionnaireResponse="questionnaireResponse"
          :questionnaire="questionnaire"
          :valueSets="valueSets"
          :baseUrl="baseUrl"
          :primary="primary"
          :secondary="secondary"
          :danger="danger"
          :language="language"
          @next="countUp"
          @removeRequiredAnswer="removeRequiredQuestionEvent($event)"
          @addRequiredAnswer="addRequiredQuestionEvent($event)"
          @answer="relayAnswer($event)"
        ></component>
      </div>
      <div v-if="!spinner.loading" class="spacer"></div>
      <div v-if="!spinner.loading && language" class="button-container">
        <!-- Button Back -->
        <button type="button" class="btn button btn-outline-primary btn-lg" v-on:click="countDown" v-if="(!this.editMode && this.count !== 0) || (this.enableReturn && this.count === 0)">
          {{ language.back }}
        </button>
        <button type="button" class="btn button btn-outline-secondary btn-lg" disabled v-if="this.editMode || (this.count === 0 && !this.enableReturn)">
          {{ language.back }}
        </button>
        <!-- Button Next -->
        <span>
          <button type="button" class="button btn-primary btn-lg" v-on:click="countUp" v-if="count <= itemList.length - 1 && !disabled && !this.editMode">
            {{ language.next }}
          </button>
          <button type="button" class="button btn-secondary btn-lg" disabled v-if="count < itemList.length && disabled">
            {{ language.next }}
          </button>
          <button type="button" class="button btn-primary btn-lg" v-on:click="goToSummary()" v-if="this.editMode && !disabled">
            {{ language.accept }}
          </button>
        </span>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.center-vertical {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
}

@mixin basic-button {
  border-radius: 10px;
  border-width: 2.2px;
  font-size: 1.2em;
  padding-left: 1.2em;
  padding-right: 1.2em;
  min-height: 1.8em;
  font-weight: bold;
  cursor: pointer;
}

.button {
  @include basic-button;
  align-self: center;
}

.title {
  font-size: 1.15em;
  font-weight: 700;
}

.color-grey {
  color: #6a6a6a;
}

.card {
  min-height: calc(100vh - 200px);
}

.column {
  display: flex;
  flex-direction: column;
}

.spacer {
  flex: 1;
}

.icon-container {
  display: flex;
  justify-content: flex-end;
}

.icon-style {
  font-size: 28px;
  margin-right: 15px;
}

.progress-counter {
  border-radius: 12px;
  font-size: 1.3em;
  padding-right: 1.2em;
  min-height: 1.8em;
  align-self: center;
}
.modal-button-container {
  padding: 12px;
}
.button-container {
  display: flex;
  justify-content: space-between;
  padding: 12px;
}
</style>

<script>
import integerQuestion from "./../../components/questions/IntegerQuestion.vue";
import decimalQuestion from "./../../components/questions/DecimalQuestion.vue";
import dateTimeQuestion from "./../../components/questions/DateTimeQuestion.vue";
import dateQuestion from "./../../components/questions/DateQuestion.vue";
import timeQuestion from "./../../components/questions/TimeQuestion.vue";
import textQuestion from "./../../components/questions/TextQuestion.vue";
import urlQuestion from "./../../components/questions/UrlQuestion.vue";
import choiceQuestion from "./../../components/questions/ChoiceQuestion.vue";
import stringQuestion from "./../../components/questions/StringQuestion.vue";
import booleanQuestion from "./../../components/questions/BooleanQuestion.vue";
import groupQuestion from "./../../components/questions/GroupQuestion.vue";
import displayQuestion from "./../../components/questions/DisplayQuestion.vue";
import questionnaireResponseController from "./../../util/questionnaireResponseController";
import Spinner from "vue-simple-spinner";
export default {
  props: {
    /**
     *
     */
    questionnaire: {
      type: Object,
      required: true
    },
    /**
     *
     */
    questionnaireResponse: {
      type: Object,
      default: null
    },
    /**
     *
     */
    requiredQuestionList: {
      type: Array
    },
    /**
     *
     */
    valueSets: {
      type: Array
    },
    /**
     *
     */
    baseUrl: {
      type: String
    },
    /**
     *
     */
    editMode: {
      type: Boolean,
      default: false
    },
    /**
     *
     */
    startCount: Number,
    /**
     *
     */
    lastQuestion: {
      type: Boolean,
      default: false
    },
    /**
     * Primary color
     */
    primary: {
      type: String
    },
    /**
     * Secondary color
     */
    secondary: {
      type: String
    },
    /**
     * Color used to symbolise danger
     */
    danger: {
      type: String
    },
    /**
     *
     */
    enableReturn: {
      type: Boolean,
      default: true
    },
    /**
     *
     */
    language: {
      type: Object
    },
    /**
     *
     */
    spinner: {
      type: Object
    }
  },
  data() {
    return {
      count: 0,
      itemList: [],
      questionCount: 0,
      disabled: false
    };
  },
  computed: {
    /**
     * Returns the Type of the current Question
     */
    getQuestionType() {
      return this.getQuestionFromItemList().type + "Question";
    },

    /**
     *Returns the current Question
     */
    getQuestion() {
      return this.getQuestionFromItemList();
    },

    /**
     *
     */
    getLastQuestion() {
      return this.lastQuestion;
    },

    /**
     * Returns the total Number of Required Questions
     */
    numberOfRequiredQuestions() {
      let totalNumber = 0;
      for (let i = 0; i < this.itemList.length; i++) {
        if (this.itemList[i].required) {
          totalNumber++;
        }
      }
      return totalNumber;
    },

    /**
     * Checks if not all required Question have been completed
     */
    notAllRequiredQuestionsCompleted() {
      return this.requiredQuestionList.length !== this.numberOfRequiredQuestions;
    },

    /**
     * Counts all Questions from ItemList excluding Groups
     */
    numberOfQuestions() {
      let number = 0;
      if (this.itemList) {
        for (let i = 0; i < this.itemList.length; i++) {
          if (this.itemList[i].type !== "group") {
            number++;
          }
        }
      }
      return number;
    }
  },

  methods: {
    /**
     *
     */
    goToSummary() {
      this.$emit("summary");
    },

    /**
     * Relays the Event from the question-components to the top-component
     */
    relayAnswer(object) {
      this.$emit("answer", object);
    },

    /**
     * Emits new Event to give the required Question to Parent-Component
     * to be removed from the List of answered Questions
     */
    removeRequiredQuestionEvent(question) {
      this.$emit("removeRequiredAnswer", question);
    },

    /**
     * Emits new Event to give the required Question to Parent-Component
     * to be added to the List of answered Questions
     */
    addRequiredQuestionEvent(question) {
      this.$emit("addRequiredAnswer", question);
    },

    /**
     * Returns a Question from the itemList
     */
    getQuestionFromItemList() {
      return this.itemList[this.count];
    },

    /**
     * Returns the positionnumber of the current question without groups
     */
    getQuestionPositionNumber() {
      let positionnumber = 1;
      for (let i = 0; i < this.count; i++) {
        if (this.itemList[i].type !== "group") {
          positionnumber++;
        }
      }
      return positionnumber;
    },

    /**
     * Counts up the Question-Number
     */
    countUp() {
      if (this.count < this.itemList.length - 1 && !this.disabled && this.startCount === null) {
        this.count++;
        if (this.itemList[this.count].type !== "group") {
          this.questionCount = this.getQuestionPositionNumber();
        }
      } else if (this.count === this.itemList.length - 1 && !this.disabled && this.startCount === null) {
        this.$emit("finished");
      } else if (this.startCount !== null) {
        this.$emit("finished");
      }
    },

    /**
     * Counts down the Question-Number
     */
    countDown() {
      //If count bigger than 0 and startCount is null
      if (this.count > 0 && this.startCount === null) {
        this.count--;
        //update questionPositionNumber
        if (this.itemList[this.count].type !== "group") {
          this.questionCount = this.getQuestionPositionNumber();
        } else if (this.itemList[this.count].type === "group" && this.questionCount === 1) {
          this.questionCount = 0;
        }
        //if count = 0 go back to Metadata
      } else if (this.count === 0) {
        this.$emit("return");
      }
    },

    /**
     *
     */
    setDisabled() {
      this.disabled = false;
      let currentQuestion = this.getQuestion;
      if (currentQuestion.required) {
        this.disabled = true;
        for (let i = 0; i < this.requiredQuestionList.length; i++) {
          if (
            // this.requiredQuestionList[i] === currentQuestion
            JSON.stringify(this.requiredQuestionList[i]) === JSON.stringify(currentQuestion)
          ) {
            this.disabled = false;
          }
        }
      }
    }
  },

  watch: {
    questionnaire() {
      this.itemList = questionnaireResponseController.createItemList(this.questionnaire);

      this.count = 0;
    },
    count() {
      this.setDisabled();
    },
    requiredQuestionList() {
      this.setDisabled();
    },
    questionnaireResponse() {
      this.setDisabled();
    }
  },

  created() {
    this.itemList = questionnaireResponseController.createItemList(this.questionnaire);
    //sets count if startcount was given from the summarypage through the questionnaire.view
    if (this.startCount) {
      this.count = this.startCount;
      this.questionCount = this.getQuestionPositionNumber();
    }
    if (this.getLastQuestion) {
      this.count = this.itemList.length - 1;
      this.questionCount = this.getQuestionPositionNumber();
    }

    //Resetting Lists to avoid leftovers
    // this.RESET_REQUIRED_ANSWERED_QUESTIONS_LIST();
    this.setDisabled();
    //Set questionCount to 1 if first Question is not a group
    if (this.itemList[this.count].type !== "group" && this.count === 0) {
      this.questionCount = 1;
    }
  },

  components: {
    Spinner,
    displayQuestion,
    integerQuestion,
    timeQuestion,
    dateTimeQuestion,
    decimalQuestion,
    dateQuestion,
    textQuestion,
    urlQuestion,
    choiceQuestion,
    stringQuestion,
    booleanQuestion,
    groupQuestion
  }
};
</script>