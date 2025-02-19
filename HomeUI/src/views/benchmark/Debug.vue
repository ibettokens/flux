<template>
  <div>
    <b-card>
      <h6 class="mb-1">
        Click the 'Download Debug File' button to download the Benchmark log. This may take a few minutes depending on file size.
      </h6>
      <div>
        <b-button
          id="start-download"
          v-ripple.400="'rgba(255, 255, 255, 0.15)'"
          variant="outline-primary"
          size="md"
        >
          Download Debug File
        </b-button>
        <div
          v-if="total && downloaded"
          class="d-flex"
          style="width: 300px;"
        >
          <b-card-text
            class="mt-1 mb-0 mr-auto"
          >
            {{ (downloaded / 1e6).toFixed(2) + " / " + (total / 1e6).toFixed(2) }} MB - {{ ((downloaded / total) * 100).toFixed(2) + "%" }}
          </b-card-text>
          <b-button
            v-ripple.400="'rgba(255, 255, 255, 0.15)'"
            variant="danger"
            class="btn-icon cancel-button"
            size="sm"
            @click="cancelDownload"
          >
            x
          </b-button>
        </div>
        <b-popover
          ref="popover"
          target="start-download"
          triggers="click"
          :show.sync="downloadPopoverShow"
          placement="auto"
          container="my-container"
        >
          <template #title>
            <div class="d-flex justify-content-between align-items-center">
              <span>Are You Sure?</span>
              <b-button
                v-ripple.400="'rgba(255, 255, 255, 0.15)'"
                class="close"
                variant="transparent"
                aria-label="Close"
                @click="onDownloadClose"
              >
                <span
                  class="d-inline-block text-white"
                  aria-hidden="true"
                >&times;</span>
              </b-button>
            </div>
          </template>

          <div>
            <b-button
              v-ripple.400="'rgba(255, 255, 255, 0.15)'"
              size="sm"
              variant="danger"
              class="mr-1"
              @click="onDownloadClose"
            >
              Cancel
            </b-button>
            <b-button
              v-ripple.400="'rgba(255, 255, 255, 0.15)'"
              size="sm"
              variant="primary"
              @click="onDownloadOk"
            >
              Download Debug
            </b-button>
          </div>
        </b-popover>
      </div>
    </b-card>
    <b-card>
      <h6 class="mb-1">
        Click the 'Show Debug File' button to view the last 100 lines of the Benchmark debug file.
      </h6>
      <div>
        <b-button
          id="start-tail"
          v-ripple.400="'rgba(255, 255, 255, 0.15)'"
          variant="outline-primary"
          size="md"
        >
          Show Debug File
        </b-button>
        <b-form-textarea
          v-if="callResponse.data.message"
          plaintext
          no-resize
          rows="30"
          :value="callResponse.data.message"
          class="mt-1"
        />
        <b-popover
          ref="popover"
          target="start-tail"
          triggers="click"
          :show.sync="tailPopoverShow"
          placement="auto"
          container="my-container"
        >
          <template #title>
            <div class="d-flex justify-content-between align-items-center">
              <span>Are You Sure?</span>
              <b-button
                v-ripple.400="'rgba(255, 255, 255, 0.15)'"
                class="close"
                variant="transparent"
                aria-label="Close"
                @click="onTailClose"
              >
                <span
                  class="d-inline-block text-white"
                  aria-hidden="true"
                >&times;</span>
              </b-button>
            </div>
          </template>

          <div>
            <b-button
              v-ripple.400="'rgba(255, 255, 255, 0.15)'"
              size="sm"
              variant="danger"
              class="mr-1"
              @click="onTailClose"
            >
              Cancel
            </b-button>
            <b-button
              v-ripple.400="'rgba(255, 255, 255, 0.15)'"
              size="sm"
              variant="primary"
              @click="onTailOk"
            >
              Show Debug
            </b-button>
          </div>
        </b-popover>
      </div>
    </b-card>
  </div>
</template>

<script>
import {
  BCard,
  BButton,
  BPopover,
  BFormTextarea,
  BCardText,
} from 'bootstrap-vue';
import ToastificationContent from '@core/components/toastification/ToastificationContent.vue';
import Ripple from 'vue-ripple-directive';
import BenchmarkService from '@/services/BenchmarkService';

export default {
  components: {
    BCard,
    BButton,
    BPopover,
    BFormTextarea,
    BCardText,
    // eslint-disable-next-line vue/no-unused-components
    ToastificationContent,
  },
  directives: {
    Ripple,
  },
  data() {
    return {
      downloadPopoverShow: false,
      tailPopoverShow: false,
      abortToken: {},
      downloaded: 0,
      total: 0,
      callResponse: {
        status: '',
        data: {},
      },
    };
  },
  methods: {
    cancelDownload() {
      this.abortToken.cancel('User download cancelled');
      this.downloaded = '';
      this.total = '';
    },
    onDownloadClose() {
      this.downloadPopoverShow = false;
    },
    async onDownloadOk() {
      const self = this;
      this.downloadPopoverShow = false;
      this.abortToken = BenchmarkService.cancelToken();
      const zelidauth = localStorage.getItem('zelidauth');
      const axiosConfig = {
        headers: {
          zelidauth,
        },
        responseType: 'blob',
        onDownloadProgress(progressEvent) {
          self.downloaded = progressEvent.loaded;
          self.total = progressEvent.total;
        },
        cancelToken: self.abortToken.token,
      };
      const response = await BenchmarkService.justAPI().get('/flux/benchmarkdebug', axiosConfig);
      const url = window.URL.createObjectURL(new Blob([response.data]));
      const link = document.createElement('a');
      link.href = url;
      link.setAttribute('download', 'debug.log');
      document.body.appendChild(link);
      link.click();
    },
    onTailClose() {
      this.tailPopoverShow = false;
    },
    async onTailOk() {
      this.tailPopoverShow = false;
      const zelidauth = localStorage.getItem('zelidauth');
      BenchmarkService.tailBenchmarkDebug(zelidauth)
        .then((response) => {
          if (response.data.status === 'error') {
            this.$toast({
              component: ToastificationContent,
              props: {
                title: response.data.data.message || response.data.data,
                icon: 'InfoIcon',
                variant: 'danger',
              },
            });
          } else {
            this.callResponse.status = response.data.status;
            this.callResponse.data = response.data.data;
          }
        })
        .catch((error) => {
          this.$toast({
            component: ToastificationContent,
            props: {
              title: 'Error while trying to get latest debug of Benchmark',
              icon: 'InfoIcon',
              variant: 'danger',
            },
          });
          console.log(error);
        });
    },
  },
};
</script>

<style>
.cancel-button {
  padding: 0;
  margin-top: 10px;
}
</style>
