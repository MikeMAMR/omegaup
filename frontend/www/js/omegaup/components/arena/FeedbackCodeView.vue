<template>
  <div>
    <div class="container-fluid" data-feedback-code-mirror>
      <textarea v-show="false" ref="cm-editor" v-model="value"></textarea>
    </div>
    <div class="container-fluid text-right py-2">
      <button
        class="btn btn-primary mx-2"
        :disabled="!numberOfComments"
        @click.prevent="saveFeedbackList"
      >
        {{ T.submissionSendFeedback }}
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import { Vue, Component, Prop, Ref } from 'vue-property-decorator';
import T from '../../lang';
import CodeMirror from 'codemirror';
import { EditorOptions, languageModeMap, modeList } from './CodeView.vue';
import Feedback, { ArenaCourseFeedback, FeedbackStatus } from './Feedback.vue';

const FeedbackClass = Vue.extend(Feedback);

for (const mode of modeList) {
  require(`codemirror/mode/${mode}/${mode}.js`);
}

@Component({
  components: {},
})
export default class FeedbackCodeView extends Vue {
  @Prop() language!: string;
  @Prop() value!: string;
  @Prop({ default: () => new Map<number, ArenaCourseFeedback>() })
  feedbackMap!: Map<number, ArenaCourseFeedback>;
  @Ref('cm-editor') private readonly cmEditor!: HTMLTextAreaElement;

  T = T;
  mode = languageModeMap[this.language] ?? languageModeMap['cpp17-gcc'];
  mapChangeTracker = 1;

  get feedbackList(): [number, ArenaCourseFeedback][] {
    if (!this.mapChangeTracker) {
      throw new Error('unreachable code');
    }
    return Array.from(this.feedbackMap);
  }

  get numberOfComments(): number {
    return this.feedbackList.length;
  }

  get editorOptions(): EditorOptions {
    return {
      tabSize: 2,
      lineNumbers: true,
      mode: this.mode,
      readOnly: true,
      gutters: ['CodeMirror-linenumbers', 'breakpoints'],
    };
  }

  mounted() {
    const editor = CodeMirror.fromTextArea(this.cmEditor, this.editorOptions);

    editor.on(
      'gutterClick',
      (editor: CodeMirror.Editor, lineNumber: number) => {
        if (editor.lineInfo(lineNumber).widgets?.length ?? 0 > 0) {
          // There's already a widget in this line, so avoid creating another one.
          return;
        }

        const feedbackForm = new FeedbackClass({
          propsData: {
            feedback: {
              text: null,
              lineNumber,
              status: FeedbackStatus.New,
            },
          },
        });
        feedbackForm.$mount();
        const lineWidget = editor.addLineWidget(
          lineNumber,
          feedbackForm.$el as HTMLElement,
          {
            className: 'px-2',
          },
        );

        feedbackForm.$on('submit', (feedback: ArenaCourseFeedback) => {
          Vue.nextTick(() => {
            // Now that the DOM has changed, we need to tell CodeMirror to
            // recalculate the height of the line widget so that it knows which
            // y coordinate corresponds to which line.
            lineWidget.changed();
          });
          this.setFeedback(feedback);
        });

        feedbackForm.$on('cancel', () => {
          editor.removeLineWidget(lineWidget);
          feedbackForm.$destroy();
        });

        feedbackForm.$on('delete', (feedback: ArenaCourseFeedback) => {
          this.deleteFeedback({ lineNumber: feedback.lineNumber });
          editor.removeLineWidget(lineWidget);
          feedbackForm.$destroy();
        });

        Vue.nextTick(() => {
          // Now that the DOM has changed, we need to tell CodeMirror to
          // recalculate the height of the line widget so that it knows which y
          // coordinate corresponds to which line.
          lineWidget.changed();
        });
      },
    );
  }

  setFeedback({
    lineNumber,
    text = null,
    status,
  }: {
    lineNumber: number;
    text: string | null;
    status: FeedbackStatus;
  }): void {
    this.feedbackMap.set(lineNumber, {
      lineNumber,
      text,
      status,
    });
    this.mapChangeTracker++;
  }

  deleteFeedback({ lineNumber }: { lineNumber: number }): void {
    this.feedbackMap.delete(lineNumber);
    this.mapChangeTracker++;
  }

  saveFeedbackList(): void {
    this.$emit(
      'save-feedback-list',
      this.feedbackList.map(([lineNumber, feedback]) => ({
        lineNumber,
        feedback: feedback.text,
      })),
    );
  }
}
</script>

<style lang="scss">
@import '../../../../sass/main.scss';
@import 'https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.2.0/css/all.css';

[data-feedback-code-mirror] {
  height: auto;

  .cm-s-default {
    height: 95%;

    .CodeMirror-sizer {
      min-height: 100% !important;
    }

    .CodeMirror-linenumber {
      width: 4em !important;
      cursor: pointer;
    }

    .CodeMirror-linenumber::after {
      font: var(--fa-font-solid);
      width: 1em;
      content: '';
      display: inline-block;
      vertical-align: middle;
      font-weight: 900;
      color: var(--btn-ok-background-color);
      font-size: x-large;
    }

    .CodeMirror-linenumber:hover::after {
      content: '\f0fe';
    }

    .CodeMirror {
      height: auto;

      .CodeMirror-scroll {
        height: auto;
      }
    }
  }
}
</style>
