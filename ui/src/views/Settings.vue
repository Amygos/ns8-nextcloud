<template>
  <div class="bx--grid bx--grid--full-width">
    <div class="bx--row">
      <div class="bx--col-lg-16 page-title">
        <h2>{{ $t("settings.title") }}</h2>
      </div>
    </div>
    <!-- sample settings page -->
    <div v-if="error.getConfiguration" class="bx--row">
      <div class="bx--col">
        <NsInlineNotification
          kind="error"
          :title="$t('action.get-configuration')"
          :description="error.getConfiguration"
          :showCloseButton="false"
        />
      </div>
    </div>
    <div v-if="!loading.settings && running && !installed" class="bx--row">
      <div class="bx--col-lg-16">
        <cv-toast-notification
          :title="$t('settings.first_config_title')"
          :sub-title="$t('settings.first_config_body')"
          :caption="nextcloud_link"
          :low-contrast="style.lowContrast"
          :hide-close-button="style.hideClose"
        ></cv-toast-notification>
      </div>
    </div>
    <div class="bx--row">
      <div class="bx--col-lg-16">
        <cv-tile :light="true">
          <cv-form @submit.prevent="saveSettings">
            <div v-if="error.configureModule" class="bx--row">
              <div class="bx--col">
                <NsInlineNotification
                  kind="error"
                  :title="$t('action.configure-module')"
                  :description="error.configureModule"
                  :showCloseButton="false"
                />
              </div>
            </div>
            <cv-text-input
              :label="$t('settings.host')"
              :placeholder="default_host"
              v-model.trim="host"
              class="mg-bottom"
              :invalid-message="$t(error.host)"
              :disabled="loading.settings"
              ref="host"
            >
            </cv-text-input>
            <cv-toggle
              value="letsEncrypt"
              :label="$t('settings.lets_encrypt')"
              v-model="isLetsEncryptEnabled"
              :disabled="loading.settings"
              class="mg-bottom"
            >
              <template slot="text-left">{{
                $t("settings.disabled")
              }}</template>
              <template slot="text-right">{{
                $t("settings.enabled")
              }}</template>
            </cv-toggle>
            <cv-combo-box
              v-if="installed"
              v-model="domain"
              :options="domains"
              auto-highlight
              :title="$t('settings.domain')"
              :invalid-message="$t(error.listUserDomains)"
              :disabled="loading.settings || loading.domains"
              :label="$t('settings.no_domain')"
              light
              ref="domain"
            >
            </cv-combo-box>
            <NsButton
              kind="primary"
              :icon="Save20"
              :loading="loading.settings"
              :disabled="loading.settings"
              >{{ $t("settings.save") }}</NsButton
            >
          </cv-form>
        </cv-tile>
      </div>
    </div>
  </div>
</template>

<script>
import to from "await-to-js";
import { mapState } from "vuex";
import {
  QueryParamService,
  UtilService,
  TaskService,
  IconService,
} from "@nethserver/ns8-ui-lib";

export default {
  name: "Settings",
  mixins: [TaskService, IconService, UtilService, QueryParamService],
  pageTitle() {
    return this.$t("settings.title") + " - " + this.appName;
  },
  data() {
    return {
      q: {
        page: "settings",
      },
      urlCheckInterval: null,
      loading: {
        settings: true,
        domains: true,
      },
      error: {
        getConfiguration: "",
        configureModule: "",
        listUserDomains: "",
      },
      style: {
        lowContrast: true,
        hideClose: true,
      },
      host: "",
      default_host: "",
      isLetsEncryptEnabled: false,
      domain: "",
      domains: [
        {
          name: "nodomain",
          label: this.$t("settings.no_domain"),
          value: "",
        },
      ],
      installed: false,
      running: false,
      nextcloud_link: "",
    };
  },
  computed: {
    ...mapState(["instanceName", "core", "appName"]),
  },
  created() {
    this.getConfiguration();
    this.listUserDomains();
  },
  beforeRouteEnter(to, from, next) {
    next((vm) => {
      vm.watchQueryData(vm);
      vm.urlCheckInterval = vm.initUrlBindingForApp(vm, vm.q.page);
    });
  },
  beforeRouteLeave(to, from, next) {
    clearInterval(this.urlCheckInterval);
    next();
  },
  methods: {
    async getConfiguration() {
      this.loading.settings = true;
      this.error.getConfiguration = "";
      const taskAction = "get-configuration";

      // register to task error
      this.core.$root.$off(taskAction + "-aborted");
      this.core.$root.$once(
        taskAction + "-aborted",
        this.getConfigurationAborted
      );

      // register to task completion
      this.core.$root.$off(taskAction + "-completed");
      this.core.$root.$once(
        taskAction + "-completed",
        this.getConfigurationCompleted
      );

      const res = await to(
        this.createModuleTaskForApp(this.instanceName, {
          action: taskAction,
          extra: {
            title: this.$t("action." + taskAction),
            isNotificationHidden: true,
          },
        })
      );
      const err = res[0];
      if (err) {
        console.error(`error creating task ${taskAction}`, err);
        this.error.getConfiguration = this.getErrorMessage(err);
        this.loading.settings = false;
        return;
      }
    },
    getConfigurationAborted(taskResult, taskContext) {
      console.error(`${taskContext.action} aborted`, taskResult);
      this.error.getConfiguration = this.core.$t("error.generic_error");
      this.loading.settings = false;
    },
    async listUserDomains() {
      this.loading.domains = true;
      this.error.listUserDomains = "";
      const taskAction = "list-user-domains";

      // register to task error
      this.core.$root.$off(taskAction + "-aborted");
      this.core.$root.$once(
        taskAction + "-aborted",
        this.listUserDomainsAborted
      );

      // register to task completion
      this.core.$root.$off(taskAction + "-completed");
      this.core.$root.$once(
        taskAction + "-completed",
        this.listUserDomainsCompleted
      );

      const res = await to(
        this.createClusterTaskForApp({
          action: taskAction,
          extra: {
            title: this.$t("action." + taskAction),
            isNotificationHidden: true,
          },
        })
      );
      const err = res[0];
      if (err) {
        console.error(`error creating task ${taskAction}`, err);
        this.error.listUserDomains = this.getErrorMessage(err);
        this.loading.domains = false;
        return;
      }
    },
    listUserDomainsAborted(taskResult, taskContext) {
      console.error(`${taskContext.action} aborted`, taskResult);
      this.error.getConfiguration = this.core.$t("error.generic_error");
      this.loading.domains = false;
    },
    validateSaveSettings() {
      this.clearErrors(this);
      let isValidationOk = true;
      return isValidationOk;
    },
    saveSettingsValidationFailed(validationErrors) {
      this.loading.settings = false;
      for (const validationError of validationErrors) {
        const param = validationError.parameter;
        // set i18n error message
        this.error[param] = "settings." + validationError.error;
      }
    },
    async saveSettings() {
      const isValidationOk = this.validateSaveSettings();
      if (!isValidationOk) {
        return;
      }
      this.loading.settings = true;
      const taskAction = "configure-module";

      // register to task error
      this.core.$root.$off(taskAction + "-aborted");
      this.core.$root.$once(taskAction + "-aborted", this.saveSettingsAborted);

      // register to task validation
      this.core.$root.$off(taskAction + "-validation-failed");
      this.core.$root.$once(
        taskAction + "-validation-failed",
        this.saveSettingsValidationFailed
      );

      // register to task completion
      this.core.$root.$off(taskAction + "-completed");
      this.core.$root.$once(
        taskAction + "-completed",
        this.saveSettingsCompleted
      );

      const res = await to(
        this.createModuleTaskForApp(this.instanceName, {
          action: taskAction,
          data: {
            host: this.host,
            lets_encrypt: this.isLetsEncryptEnabled,
            domain: this.domain,
          },
          extra: {
            title: this.$t("settings.instance_configuration", {
              instance: this.instanceName,
            }),
            description: this.$t("settings.configuring"),
          },
        })
      );
      const err = res[0];
      if (err) {
        console.error(`error creating task ${taskAction}`, err);
        this.error.configureModule = this.getErrorMessage(err);
        this.loading.settings = false;
        return;
      }
    },
    saveSettingsAborted(taskResult, taskContext) {
      console.error(`${taskContext.action} aborted`, taskResult);
      this.error.getConfiguration = this.core.$t("error.generic_error");
      this.loading.settings = false;
    },
    getConfigurationCompleted(taskContext, taskResult) {
      const config = taskResult.output;
      this.host = config.host;
      this.isLetsEncryptEnabled = config.lets_encrypt;
      this.default_host = "nextcloud." + config.default_domain;
      this.running = config.running;
      this.installed = config.installed;
      this.loading.settings = false;
      if (this.host) {
        this.nextcloud_link = `${this.$t(
          "settings.open_link"
        )}: <a href='https://${this.host}' target='_blank'>${this.host}</a>`;
      }
      this.focusElement("host");
    },
    listUserDomainsCompleted(taskContext, taskResult) {
      const domains = taskResult.output;

      domains["domains"].forEach((element) => {
        const option = {
          name: element["name"],
          label: element["name"],
          value: element["name"],
        };
        this.domains.push(option);
      });
      this.loading.domains = false;
    },
    saveSettingsCompleted() {
      this.loading.settings = false;
      // reload configuration
      this.getConfiguration();
    },
  },
};
</script>

<style scoped lang="scss">
@import "../styles/carbon-utils";
</style>
