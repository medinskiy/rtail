<template>
  <div
    :style="`font-size: ${fontSize}px;`"
    class="stream-content">
    <md-toolbar
      class="md-dense">
      <md-button
        class="md-icon-button"
        @click="showTimestamp = !showTimestamp">
        <md-icon>{{ showTimestamp ? 'timer_off' : 'timer' }}</md-icon>
      </md-button>
      <span class="md-caption">{{ streamId }}</span>
      <div class="md-toolbar-section-end stream-content__header-buttons">
        <md-field
          class="stream-content__filter"
          md-clearable>
          <label>Stream filter</label>
          <md-input v-model="streamLineFilter"/>
        </md-field>
        <md-button
          v-if="!activeStreamIsLast(streamId)"
          class="md-icon-button"
          @click="streamMoveDown(streamId)">
          <md-icon>keyboard_arrow_down</md-icon>
        </md-button>
        <md-button
          v-if="!activeStreamIsFirst(streamId)"
          class="md-icon-button"
          @click="streamMoveUp(streamId)">
          <md-icon>keyboard_arrow_up</md-icon>
        </md-button>
        <md-button
          v-if="!activeStreamIsFirst(streamId) || !activeStreamIsLast(streamId)"
          class="md-icon-button"
          @click="streamClose(streamId)">
          <md-icon>clear</md-icon>
        </md-button>
      </div>
    </md-toolbar>
    <md-content class="backlog">
      <md-empty-state
        v-if="!orderedBacklog.length"
        :md-label="`Nothing for: ${streamId}`"
        class="md-accent"
        md-icon="remove_circle_outline"
        md-description="Select other stream or check your r-tail client" />
      <div
        v-if="orderedBacklog.length"
        class="backlog__wrapper">
        <div
          v-for="line in backlogFilter(orderedBacklog)"
          :key="line.uid"
          class="backlog__line-container hljs">
          <div
            v-if="showTimestamp"
            class="backlog__line-timestamp">
            {{ line.timestamp | dateFormat }}
          </div>
          <div
            v-highlightjs="line.html"
            v-if="line.isJSON"
            class="backlog__line-content backlog__line-content-highlight">
            <pre><code class="json"/></pre>
          </div>
          <div
            v-if="!line.isJSON"
            class="backlog__line-content backlog__line-content-text"
            v-html="line.html"/>
        </div>
      </div>
    </md-content>
  </div>
</template>

<script>
import fecha from 'fecha';
import { mapState, mapGetters } from 'vuex';

export default {
  components: {},
  filters: {
    dateFormat(ts) {
      return fecha.format(new Date(ts), 'MM/DD/YY hh:mm:ss');
    },
  },
  props: {
    streamId: {
      type: String,
      required: true,
    },
  },
  data: () => ({
    showTimestamp: true,
    streamLineFilter: '',
  }),
  computed: {
    ...mapGetters([
      'activeStreamIsLast',
      'activeStreamIsFirst',
    ]),
    ...mapState({
      isloadingComplete: state => state.isStreamsLoaded,
      fontSize: state => state.settings.fontSize,
    }),
    orderedBacklog(state) {
      return this.$store.getters.backlogDESC(this.streamId);
    },
  },
  mounted() {
    this.$nextTick(function () {
      this.$socket.emit('streamSubscribe', this.$props.streamId);
    });
  },
  beforeDestroy() {
    this.$socket.emit('streamUnsubscribe', this.$props.streamId);
  },
  methods: {
    streamMoveUp(streamId) {
      this.$store.commit('activeStreamMove', { streamId, direction: 1 });
    },
    streamMoveDown(streamId) {
      this.$store.commit('activeStreamMove', { streamId, direction: -1 });
    },
    streamClose(streamId) {
      this.$store.commit('activeStreamClose', { streamId });
    },
    backlogFilter(backlog = []) {
      const regExp = new RegExp(this.streamLineFilter);
      return backlog.filter(line => regExp.test(line.content));
    },
  },
};
</script>

<style>
.backlog {
  max-height: calc(100% - 48px);
  height: 100%;
  overflow: auto;
}
.backlog__line-container.hljs {
  padding: 5px 0;
}
.backlog__line-container {
  display: flex;
  flex-wrap: nowrap;
  align-items: center;
  border-bottom: 1px solid rgba(0, 0, 0, 0.2);
}
.backlog__line-timestamp {
  width: 115px;
  font-size: 0.8em;
  padding-left: 5px;
  flex-shrink: 0;
}
.backlog__line-container {
  line-height: 1.1em;

  pre {
    margin: 0;
  }
}
.backlog__line-content {
  padding-left: 5px;
  word-break: break-word;
}
.stream-content__filter {
  max-width: 200px;
}
</style>
