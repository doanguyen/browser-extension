<template>
  <div class style="width: 450px">
    <div class="text-center mt-2" v-if="loading">
      <b-spinner type="grow" variant="primary" label="Spinning"></b-spinner>
    </div>

    <div class="text-center mt-3">
      <a href="https://www.simplelogin.io" target="_blank">
        <img src="/images/horizontal-logo.svg" />
      </a>
      <hr />
    </div>

    <!-- No API Key set: Setup screen -->
    <div v-if="apiKey == ''" class="p-2 container">
      <h1 class="h5">
        Welcome to
        <a href="https://simplelogin.io" target="_blank">SimpleLogin ↗</a>, the most powerful email alias solution!
      </h1>

      <p>To get started, please follow these 3 simple steps:</p>

      <div class="mb-2">
        <span class="badge badge-primary badge-pill">1</span>
        Create your SimpleLogin account
        <a
          href="https://app.simplelogin.io/auth/register"
          target="_blank"
        >here</a>
        if this is not already done.
      </div>

      <div class="mb-2">
        <span class="badge badge-primary badge-pill">2</span>
        Create and copy your
        <em>API Key</em>
        <a href="https://app.simplelogin.io/dashboard/api_key" target="_blank">here</a>.
      </div>

      <div class="mb-2">
        <span class="badge badge-primary badge-pill">3</span>
        Paste the
        <em>API Key</em> here 👇🏽
      </div>

      <input
        v-model="apiInput"
        v-on:keyup.enter="save"
        placeholder="API Key"
        autofocus
        class="form-control mt-3 w-100"
      />

      <button @click="save" class="btn btn-primary btn-block mt-2">Set API Key</button>
    </div>

    <!-- API Key is set -->
    <div v-else>
      <!-- Alias option page -->
      <div v-if="!optionsReady && loading" class="container text-center">Please wait ...</div>

      <div v-if="optionsReady && newAlias == ''" class="container">
        <div v-if="hasRecommendation" class="text-center">
          <span class="text-success">{{ recommendation.alias }}</span>
          <button
            v-if="recommendation.alias"
            v-clipboard="() => recommendation.alias"
            v-clipboard:success="clipboardSuccessHandler"
            v-clipboard:error="clipboardErrorHandler"
            class="btn btn-success btn-sm"
          >Copy</button>
          <br />
          <div class="small-text">recommended, already used on this website.</div>

          <hr />
        </div>

        <p class="font-weight-bold text-center mb-2">Email Alias</p>
        <div>
          <form @submit.prevent="createCustomAlias">
            <div class="row mb-2">
              <div class="col" style="padding-right: 0">
                <input
                  v-model="aliasPrefix"
                  class="form-control"
                  pattern="[0-9|A-Z|a-z|-|_]{1,}"
                  title="Only letter, number, dash (-), underscore (_) can be used in alias prefix."
                  placeholder="alias prefix"
                  autofocus
                  required
                />
              </div>

              <div class="col align-self-center" style="padding-left: 5px">
                <select
                  v-if="custom.suffixes.length > 1"
                  v-model="aliasSuffix"
                  class="form-control"
                >
                  <option v-for="suffix in custom.suffixes" v-bind:key="suffix">{{ suffix }}</option>
                </select>

                <span v-if="custom.suffixes.length == 1">{{custom.suffixes[0]}}</span>
              </div>
            </div>

            <div
              class="small-text mb-1"
              v-if="aliasPrefix"
            >Alias is autofilled by the current website name, please feel free to change it.</div>

            <button
              :disabled="loading || !canCreateCustom"
              class="btn btn-primary btn-block mt-2"
            >Create</button>
          </form>
        </div>

        <div v-if="!canCreateCustom">
          <p class="text-danger" style="font-size: 14px">
            You have created 3 email aliases in free plan, please
            <a
              href="https://app.simplelogin.io/dashboard/pricing"
              target="_blank"
            >upgrade</a> or reuse the alias.
          </p>
        </div>
        <hr />

        <div v-if="existing.length > 0" class="text-center">
          <p class="font-weight-bold">Or use an existing alias</p>
          <div v-for="alias in existing" v-bind:key="alias">
            <span class="text-info">{{ alias }}</span>
            <button
              v-if="alias"
              v-clipboard="() => alias"
              v-clipboard:success="clipboardSuccessHandler"
              v-clipboard:error="clipboardErrorHandler"
              class="btn btn-success btn-sm copy-btn"
            >Copy</button>
          </div>
        </div>
      </div>

      <div v-if="newAlias != ''" class="text-center">
        <p class="font-weight-bold">Alias is created</p>
        <span class="text-info">{{ newAlias }}</span>
        <button
          v-if="newAlias"
          v-clipboard="() => newAlias"
          v-clipboard:success="clipboardSuccessHandler"
          v-clipboard:error="clipboardErrorHandler"
          class="btn btn-success btn-sm"
        >Copy</button>

        <br />

        <button @click="backToOptionPage" class="btn btn-secondary mt-3">&lt; Back</button>
      </div>

      <!-- Footer -->
      <hr />
      <div>
        <a
          href="https://app.simplelogin.io/dashboard/"
          target="_blank"
          class="btn btn-sm btn-link float-left"
        >Manage Aliases</a>
        <button @click="reset" class="btn btn-sm btn-link float-right">Logout</button>
      </div>
    </div>
  </div>
</template>

<script>
import "bootstrap/dist/css/bootstrap.css";
import "bootstrap-vue/dist/bootstrap-vue.css";

// Local API
// const API = "http://localhost:7777/api";

const API = "https://app.simplelogin.io/api";

function getInitialData() {
  return {
    loading: false,

    // API key
    apiKey: "",
    apiInput: "",

    // hostName obtained from chrome tabs query
    hostName: "",

    // new alias is saved here: a new alias screen will be shown
    newAlias: "",

    // only show options when GET /alias/options returns
    optionsReady: false,

    // for recommendation section
    hasRecommendation: false,
    recommendation: {},

    // for custom
    custom: {},
    aliasPrefix: "",
    aliasSuffix: "",

    canCreateCustom: false,

    existing: []
  };
}

export default {
  data() {
    return getInitialData();
  },
  async mounted() {
    let that = this;
    chrome.storage.sync.get("apiKey", async function(data) {
      that.apiKey = data.apiKey || "";
      that.apiInput = that.apiKey || "";

      that.hostName = await that.getHostName();

      if (that.apiKey != "") that.getAliasOptions();
    });
  },
  methods: {
    async save() {
      let that = this;
      if (this.apiInput === "") {
        that.showError("API Key cannot be empty");
        return;
      }

      chrome.storage.sync.set({ apiKey: this.apiInput }, function() {
        chrome.storage.sync.get("apiKey", function(data) {
          that.$toasted.show("API Key set successfully", { type: "success" });
          that.apiKey = data.apiKey;

          that.getAliasOptions();
        });
      });
    },
    async reset() {
      let that = this;
      chrome.storage.sync.set({ apiKey: "" }, async function() {
        that.apiKey = "";
        that.apiInput = "";

        Object.assign(that.$data, getInitialData());
        that.hostName = await that.getHostName();
      });
    },
    async backToOptionPage() {
      this.newAlias = "";
      this.optionsReady = false;
      await this.getAliasOptions();
    },
    async getAliasOptions() {
      let that = this;
      that.loading = true;

      let res = await fetch(API + "/alias/options?hostname=" + that.hostName, {
        method: "GET",
        headers: {
          "Content-Type": "application/json",
          Authentication: this.apiKey
        }
      });

      let json = await res.json();

      if (res.status == 401) {
        that.showError(
          "Invalid API Key. Please logout and re-setup the API Key"
        );
        that.loading = false;
        return;
      } else if (res.status >= 500) {
        that.showError("Unknown error. We are sorry for this inconvenience!");
        that.loading = false;
        return;
      }

      if (json.recommendation !== undefined) {
        that.hasRecommendation = true;
        that.recommendation = json.recommendation || {};
      }

      if (json.custom !== undefined) {
        that.custom = json.custom;
        that.aliasPrefix = that.custom.suggestion;
        that.aliasSuffix = that.custom.suffixes[0];
      }

      that.canCreateCustom = json.can_create_custom;
      that.existing = json.existing;

      that.optionsReady = true;

      that.loading = false;
    },

    async createCustomAlias() {
      let that = this;
      that.loading = true;

      let res = await fetch(
        API + "/alias/custom/new?hostname=" + that.hostName,
        {
          method: "POST",
          body: JSON.stringify({
            alias_prefix: that.aliasPrefix,
            alias_suffix: that.aliasSuffix
          }),
          headers: {
            "Content-Type": "application/json",
            Authentication: this.apiKey
          }
        }
      );

      let json = await res.json();
      that.loading = false;
      if (res.status == 201) {
        that.newAlias = json.alias;
      } else {
        that.showError(json.error);
      }
    },

    showError(msg) {
      this.$toasted.show(msg, {
        type: "error",
        duration: null,
        action: {
          text: "x",
          onClick: (e, toastObject) => {
            toastObject.goAway(0);
          }
        }
      });
    },

    // Clipboard
    clipboardSuccessHandler({ value, event }) {
      this.$toasted.show(value + " copied to clipboard", {
        type: "success",
        duration: 2500
      });
    },

    clipboardErrorHandler({ value, event }) {
      console.log("error", value);
    },

    async getHostName() {
      try {
        var result = await this.$browser.tabs.query({
          active: true,
          currentWindow: true
        });
        var url = new URL(result[0].url);
        return url.hostname;
      } catch (error) {
        console.log(error);
      }
    }
  }
};
</script>

<style lang="scss" scoped>
p {
  font-size: 20px;
}

em {
  font-style: normal;
  background-color: #ffff00;
}

.small-text {
  font-size: 12px;
  font-weight: lighter;
}

.copy-btn {
  font-size: 0.6rem;
  line-height: 0.75;
}
</style>
