<template>
  <div>
    <v-card class="mb-5" :color="$vuetify.dark ? '' : 'grey lighten-4'">
      <v-card-text>
        <v-form v-model="valid">
          <!-- 类型 -->
          <v-text-field
            :label="$t('settings.backup.server.editor.type')"
            :placeholder="$t('settings.backup.server.editor.type')"
            disabled
            :value="type"
          ></v-text-field>

          <!-- 名称 -->
          <v-text-field
            v-model="option.name"
            :label="$t('settings.backup.server.editor.name')"
            :placeholder="$t('settings.backup.server.editor.name')"
            required
            :rules="rules.require"
            ref="name"
          ></v-text-field>

          <!-- 地址 -->
          <v-text-field
            v-model="option.address"
            :label="$t('settings.backup.server.editor.address')"
            :placeholder="$t('settings.backup.server.editor.address')"
            required
            :rules="[rules.url]"
          ></v-text-field>

          <!-- 授权码 -->
          <template v-if="type==='OWSS'">
            <v-text-field
              v-model="option.authCode"
              :label="$t('settings.backup.server.editor.authCode')"
              :placeholder="$t('settings.backup.server.editor.authCode')"
              required
              :rules="rules.require"
              :type="showAuthCode ? 'text' : 'password'"
            >
              <template v-slot:append>
                <v-icon
                  @click="showAuthCode = !showAuthCode"
                >{{showAuthCode ? 'visibility_off' : 'visibility'}}</v-icon>
                <v-btn
                  flat
                  small
                  color="primary"
                  @click="applyAuthCode"
                >{{ $t('settings.backup.server.editor.applyAuthCode') }}</v-btn>
              </template>
            </v-text-field>
          </template>

          <template v-else>
            <!-- 登录名 -->
            <v-text-field
              v-model="option.loginName"
              :label="$t('settings.backup.server.editor.loginName')"
              :placeholder="$t('settings.backup.server.editor.loginName')"
              required
              :rules="rules.require"
            ></v-text-field>

            <!-- 密码 -->
            <v-text-field
              v-model="option.loginPwd"
              :label="$t('settings.backup.server.editor.loginPwd')"
              :placeholder="$t('settings.backup.server.editor.loginPwd')"
              required
              :rules="rules.require"
              :type="showLoginPwd ? 'text' : 'password'"
            >
              <template v-slot:append>
                <v-icon
                  @click="showLoginPwd = !showLoginPwd"
                >{{showLoginPwd ? 'visibility_off' : 'visibility'}}</v-icon>
              </template>
            </v-text-field>

            <v-switch :label="$t('settings.backup.server.editor.digest')" v-model="option.digest"></v-switch>
          </template>
        </v-form>

        <v-btn
          flat
          block
          :color="testButtonColor"
          :loading="testing"
          :disabled="testing || !valid"
          @click="testServerConnectivity"
        >
          <v-icon class="mr-2">{{ testButtonIcon }}</v-icon>
          {{ successMsg || errorMsg || $t('settings.downloadClients.editor.test') }}
        </v-btn>
      </v-card-text>
    </v-card>
    <v-snackbar v-model="haveError" absolute top :timeout="3000" color="error">{{ errorMsg }}</v-snackbar>
    <v-snackbar
      v-model="haveSuccess"
      absolute
      bottom
      :timeout="3000"
      color="success"
    >{{ successMsg }}</v-snackbar>
  </div>
</template>

<script lang="ts">
import md5 from "blueimp-md5";
import Vue from "vue";
import Extension from "@/service/extension";
import {
  EAction,
  DataResult,
  Dictionary,
  EBackupServerType
} from "@/interface/common";
const extension = new Extension();
export default Vue.extend({
  data() {
    return {
      showLoginPwd: false,
      showAuthCode: false,
      rules: {
        require: [(v: any) => !!v || "!"],
        url: (v: any) => {
          return (
            /^(https?):\/\/[-A-Za-z0-9+&@#/%?=~_|!:,.;\[\]]+[-A-Za-z0-9+&@#/%=~_|]$/.test(
              v
            ) || this.$t("settings.backup.server.owss.addressTip")
          );
        }
      },
      haveError: false,
      haveSuccess: false,
      successMsg: "",
      errorMsg: "",
      valid: false,
      option: {
        authCode: "",
        address: "",
        name: "",
        loginName: "",
        loginPwd: "",
        type: EBackupServerType.OWSS,
        digest: false
      } as any,
      testing: false,
      testButtonIcon: "compass_calibration",
      testButtonColor: "info",
      testButtonStatus: {
        success: "success",
        error: "error"
      },
      buttonColor: {
        default: "info",
        success: "success",
        error: "error"
      } as Dictionary<any>,
      buttonIcon: {
        default: "compass_calibration",
        success: "done",
        error: "close"
      } as Dictionary<any>
    };
  },
  props: {
    initData: Object,
    isNew: Boolean,
    type: {
      type: String,
      default: EBackupServerType.OWSS
    },
    show: Boolean
  },
  watch: {
    show() {
      if (this.show && this.$refs.name) {
        (this.$refs.name as any).focus();
      }
    },
    successMsg() {
      this.haveSuccess = this.successMsg != "";
    },
    errorMsg() {
      this.haveError = this.errorMsg != "";
    },
    option: {
      handler() {
        setTimeout(() => {
          this.$emit("change", {
            data: this.option,
            valid: this.valid
          });
        }, 100);
      },
      deep: true
    },
    type() {
      this.option = Object.assign({}, this.initData);
      this.option.type = this.type;
    },
    initData() {
      if (this.initData) {
        if (this.initData.digest === undefined) {
          this.initData.digest = false;
        }
        this.option = Object.assign(this.option, this.initData);
        this.option.type = this.type;
      }
    }
  },
  methods: {
    applyAuthCode() {
      this.successMsg = "";
      this.errorMsg = "";
      if (this.option.address) {
        $.ajax({
          url: `${this.option.address}/create`
        })
          .done(result => {
            console.log(result);
            if (result && result.data) {
              this.option.authCode = result.data;
              this.successMsg = result.data;
            } else if (result.code && result.msg) {
              this.errorMsg = result.msg + " (" + result.code + ")";
            }
          })
          .fail((jqXHR, status, text) => {
            if (
              jqXHR.responseJSON &&
              jqXHR.responseJSON.code &&
              jqXHR.responseJSON.msg
            ) {
              this.errorMsg =
                jqXHR.responseJSON.msg + " (" + jqXHR.responseJSON.code + ")";
            } else {
              this.errorMsg = status + ", " + text;
            }
          });
      }
    },

    /**
     * 测试服务器是否可用
     */
    testServerConnectivity() {
      this.successMsg = "";
      this.errorMsg = "";
      let options = Object.assign({}, this.option);
      if (!options.address) {
        this.errorMsg = this.$t(
          "settings.downloadClients.editor.testAddressError"
        ).toString();
        return;
      }
      this.testing = true;

      extension
        .sendRequest(EAction.testBackupServerConnectivity, null, options)
        .then((result: DataResult) => {
          console.log(result);
          if (result) {
            this.successMsg = this.$t(
              "settings.downloadClients.editor.testSuccess"
            ).toString();
            this.setTestButtonStatus(this.testButtonStatus.success);
          } else {
            this.errorMsg = this.$t(
              "settings.downloadClients.editor.testError"
            ).toString();
          }
          this.errorMsg &&
            this.setTestButtonStatus(this.testButtonStatus.error);
          this.testing = false;
        })
        .catch((result: DataResult) => {
          console.log(result);
          if (result && result.data && result.data.msg) {
            this.errorMsg = result.data.msg;
          } else {
            this.errorMsg = this.$t(
              "settings.downloadClients.editor.testError"
            ).toString();
          }

          this.setTestButtonStatus(this.testButtonStatus.error);
          this.testing = false;
        });
    },

    setTestButtonStatus(status: string) {
      this.testButtonIcon = this.buttonIcon[status];
      this.testButtonColor = this.buttonColor[status];
      window.setTimeout(() => {
        this.testButtonIcon = this.buttonIcon.default;
        this.testButtonColor = this.buttonColor.default;
        this.successMsg = "";
        this.errorMsg = "";
      }, 3000);
    }
  }
});
</script>
