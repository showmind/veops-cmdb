<template>
  <div class="ops-login">
    <div class="ops-login-left">
      <span>{{ $t('cs.login.loginText') }}</span>
    </div>
    <div class="ops-login-right">
      <img src="../../assets/logo_VECMDB.png" />
      <a-form
        id="formLogin"
        ref="formLogin"
        :form="form"
        @submit="handleSubmit"
        hideRequiredMark
        :colon="false">
        <a-form-item :label="$t('cs.login.username')">
          <a-input
            size="large"
            type="text"
            class="ops-input"
            v-decorator="[
              'username',
              {
                rules: [
                  { required: true, message: $t('cs.login.usernameRequired') },
                  { validator: handleUsernameOrEmail }
                ],
                validateTrigger: 'change',
              },
            ]"
          >
          </a-input>
        </a-form-item>

        <a-form-item :label="$t('cs.login.password')">
          <a-input
            size="large"
            type="password"
            autocomplete="false"
            class="ops-input"
            v-decorator="[
              'password',
              { rules: [{ required: true, message: $t('cs.login.passwordRequired') }], validateTrigger: 'blur' }
            ]"
          >
          </a-input>
        </a-form-item>

        <a-form-item>
          <a-checkbox v-decorator="['rememberMe', { valuePropName: 'checked' }]">
            {{ $t('cs.login.autoLogin') }}
          </a-checkbox>
        </a-form-item>

        <a-form-item style="margin-top:24px">
          <a-button
            size="large"
            type="primary"
            htmlType="submit"
            class="login-button"
            :loading="state.loginBtn"
            :disabled="state.loginBtn"
          >
            {{ $t('cs.login.loginBtn') }}
          </a-button>
          <a-checkbox
            v-if="hasLDAP"
            v-model="auth_with_ldap"
          >
            LDAP
          </a-checkbox>
        </a-form-item>
      </a-form>
      <template v-if="_enable_list && _enable_list.length >= 1">
        <a-divider style="font-size:14px">
          {{ $t('cs.login.otherLoginWay') }}
        </a-divider>
        <div style="text-align:center">
          <span v-for="(item, index) in _enable_list" :key="item.auth_type">
            <ops-icon :type="item.auth_type" />
            <a @click="otherLogin(item.auth_type)">{{ item.auth_type }}</a>
            <a-divider v-if="index < _enable_list.length - 1" type="vertical" />
          </span>
        </div>
      </template>
    </div>
  </div>
</template>

<script>
import md5 from 'md5'
import { mapState, mapActions } from 'vuex'
import { timeFix } from '@/utils/util'
import appConfig from '@/config/app.js'

export default {
  name: 'Login',
  data() {
    return {
      customActiveKey: 'tab1',
      loginBtn: false,
      // login type: 0 email, 1 username, 2 telephone
      loginType: 0,
      requiredTwoStepCaptcha: false,
      stepCaptchaVisible: false,
      form: this.$form.createForm(this),
      state: {
        time: 60,
        loginBtn: false,
        // login type: 0 email, 1 username, 2 telephone
        loginType: 0,
        smsSendBtn: false,
      },
      auth_with_ldap: false,
    }
  },
  computed: {
    ...mapState({ auth_enable: (state) => state?.user?.auth_enable ?? {} }),
    enable_list() {
      return this.auth_enable?.enable_list ?? []
    },
    hasLDAP() {
      return this.enable_list.some((en) => en.auth_type === 'LDAP')
    },
    _enable_list() {
      return this.enable_list.filter((en) => en.auth_type !== 'LDAP')
    },
  },
  watch: {
    hasLDAP: {
      immediate: true,
      handler(newVal) {
        this.auth_with_ldap = newVal
      },
    },
  },
  created() {},
  async mounted() {
    await this.GetAuthDataEnable()
  },
  methods: {
    ...mapActions(['Login', 'GetAuthDataEnable']),
    // handler
    handleUsernameOrEmail(rule, value, callback) {
      const { state } = this
      const regex = /^([a-zA-Z0-9_-])+@([a-zA-Z0-9_-])+((\.[a-zA-Z0-9_-]{2,3}){1,2})$/
      if (regex.test(value)) {
        state.loginType = 0
      } else {
        state.loginType = 1
      }
      callback()
    },
    handleSubmit(e) {
      e.preventDefault()
      const {
        hasLDAP,
        form: { validateFields },
        state,
        customActiveKey,
        Login,
        auth_with_ldap,
      } = this

      state.loginBtn = true

      const validateFieldsKey = customActiveKey === 'tab1' ? ['username', 'password'] : ['mobile', 'captcha']

      validateFields(validateFieldsKey, { force: true }, (err, values) => {
        if (!err) {
          const loginParams = { ...values }
          delete loginParams.username
          loginParams.username = values.username
          loginParams.password = appConfig.useEncryption ? md5(values.password) : values.password
          loginParams.auth_with_ldap = hasLDAP ? Number(auth_with_ldap) : undefined

          localStorage.setItem('ops_auth_type', '')
          Login({ userInfo: loginParams })
            .then((res) => this.loginSuccess(res))
            .finally(() => {
              state.loginBtn = false
            })
        } else {
          setTimeout(() => {
            state.loginBtn = false
          }, 600)
        }
      })
    },
    otherLogin(auth_type) {
      this.Login({ userInfo: {}, auth_type })
    },
    loginSuccess(res) {
      this.$router.push({ path: this.$route.query?.redirect ?? '/' })
      // 延迟 1 秒显示欢迎信息
      setTimeout(() => {
        this.$notification.success({
          message: this.$t('cs.login.welcomeMessage'),
          description: this.$t('cs.login.welcomeDesc', { name: timeFix() }),
        })
      }, 1000)
    },
  },
}
</script>

<style lang="less" scoped>
.ops-login {
  width: 100%;
  height: 100%;
  display: flex;
  min-width: 1000px;
  overflow-x: auto;
  .ops-login-left {
    position: relative;
    width: 50%;
    background: url('../../assets/login_bg.png') no-repeat;
    background-position: center;
    background-size: cover;
    > img {
      width: 80%;
      position: absolute;
      top: 60%;
      left: 50%;
      transform: translate(-50%, -50%);
    }
    > span {
      color: @text-color_2;
      position: absolute;
      bottom: 10%;
      left: 50%;
      transform: translateX(-50%);
      font-size: 26px;
    }
  }
  .ops-login-right {
    width: 50%;
    position: relative;
    padding: 10%;
    > img {
      width: 70%;
      margin-left: 15%;
    }
    .ops-login-right-welcome {
      text-align: center;
      color: rgba(29, 57, 196, 1);
      font-family: 'Inter';
      font-style: normal;
      font-weight: 600;
      font-size: 1.25vw;
    }
    .login-button {
      width: 100%;
    }
  }
}
</style>
