<template>
  <div class="welcome">
    <el-container style="height: 100%;">
      <el-header>
        <div style="display: flex;align-items: center;margin-top: 15px;margin-left: 10px;gap: 10px;">
          <!--img loading="lazy" alt="" src="@/assets/header/robot.png" style="width: 45px;height: 45px;" /-->
          <img loading="lazy" alt="" src="@/assets/xiaozhi-ai.png" style="height: 38px;" />
        </div>
      </el-header>
      <!--div class="login-person">
        <img loading="lazy" alt="" src="@/assets/login/login-person.png" style="width: 100%;" />
      </div-->
      <el-main style="position: relative; top: 20%; left: 0%; transform: translate(-40%, -35%);">
        <div class="login-box" @keyup.enter="login">
          <div style="display: flex;align-items: center;gap: 20px;margin-bottom: 39px;padding: 0 30px;">
            <img loading="lazy" alt="" src="@/assets/login/hi.png" style="width: 34px;height: 34px;" />
            <div class="login-text">Login</div>
            <div class="login-welcome">
              Connect AI to the physical world
            </div>
          </div>
          <div style="padding: 0 30px;">
            <!-- username login -->
            <template v-if="!isMobileLogin">
              <div class="input-box">
                <img loading="lazy" alt="" class="input-icon" src="@/assets/login/username.png" />
                <el-input v-model="form.username" placeholder="Please enter username" />
              </div>
            </template>

            <!-- mobile number login -->
            <template v-else>
              <div class="input-box">
                <div style="display: flex; align-items: center; width: 100%;">
                  <el-select v-model="form.areaCode" style="width: 220px; margin-right: 10px;">
                    <el-option v-for="item in mobileAreaList" :key="item.key" :label="`${item.name} (${item.key})`"
                      :value="item.key" />
                  </el-select>
                  <el-input v-model="form.mobile" placeholder="Please enter mobile number" />
                </div>
              </div>
            </template>

            <div class="input-box">
              <img loading="lazy" alt="" class="input-icon" src="@/assets/login/password.png" />
              <el-input v-model="form.password" placeholder="Please enter password" type="password" show-password />
            </div>
            <div style="display: flex; align-items: center; margin-top: 20px; width: 100%; gap: 10px;">
              <div class="input-box" style="width: calc(100% - 130px); margin-top: 0;">
                <img loading="lazy" alt="" class="input-icon" src="@/assets/login/shield.png" />
                <el-input v-model="form.captcha" placeholder="Please enter verification code" style="flex: 1;" />
              </div>
              <img loading="lazy" v-if="captchaUrl" :src="captchaUrl" alt="Verification code"
                style="width: 150px; height: 40px; cursor: pointer;" @click="fetchCaptcha" />
            </div>
            <div
              style="font-weight: 400;font-size: 14px;text-align: left;color: #5778ff;display: flex;justify-content: space-between;margin-top: 20px;">
              <div v-if="allowUserRegister" style="cursor: pointer;" @click="goToRegister">New user registration</div>
              <div style="cursor: pointer;" @click="goToForgetPassword" v-if="enableMobileRegister">Forgot password?</div>
            </div>
          </div>
          <div class="login-btn" @click="login">Login</div>

          <!-- Switch login method -->
          <div class="login-type-container" v-if="enableMobileRegister">
            <el-tooltip content="Mobile number login" placement="bottom">
              <el-button :type="isMobileLogin ? 'primary' : 'default'" icon="el-icon-mobile" circle
                @click="switchLoginType('mobile')"></el-button>
            </el-tooltip>
            <el-tooltip content="Username login" placement="bottom">
              <el-button :type="!isMobileLogin ? 'primary' : 'default'" icon="el-icon-user" circle
                @click="switchLoginType('username')"></el-button>
            </el-tooltip>
          </div>

          <!--div class="login-welcome">
            By logging in, you agree to the
            <div style="display: inline-block;color: #5778FF;cursor: pointer;">"User Agreement"</div>
            and
            <div style="display: inline-block;color: #5778FF;cursor: pointer;">"Privacy Policy"</div>
          </div-->
        </div>
      </el-main>
      <el-footer>
        <version-footer />
      </el-footer>
    </el-container>
  </div>
</template>

<script>
import Api from '@/apis/api';
import VersionFooter from '@/components/VersionFooter.vue';
import { getUUID, goToPage, showDanger, showSuccess, validateMobile } from '@/utils';
import { mapState } from 'vuex';

export default {
  name: 'login',
  components: {
    VersionFooter
  },
  computed: {
    ...mapState({
      allowUserRegister: state => state.pubConfig.allowUserRegister,
      enableMobileRegister: state => state.pubConfig.enableMobileRegister,
      mobileAreaList: state => state.pubConfig.mobileAreaList
    })
  },
  data() {
    return {
      activeName: "username",
      form: {
        username: '',
        password: '',
        captcha: '',
        captchaId: '',
        areaCode: '+86',
        mobile: ''
      },
      captchaUuid: '',
      captchaUrl: '',
      isMobileLogin: false
    }
  },
  mounted() {
    this.fetchCaptcha();
    this.$store.dispatch('fetchPubConfig').then(() => {
      // default login method from config
      this.isMobileLogin = this.enableMobileRegister;
    });
  },
  methods: {
    fetchCaptcha() {
      if (this.$store.getters.getToken) {
        if (this.$route.path !== '/home') {
          this.$router.push('/home')
        }
      } else {
        this.captchaUuid = getUUID();

        Api.user.getCaptcha(this.captchaUuid, (res) => {
          if (res.status === 200) {
            const blob = new Blob([res.data], { type: res.data.type });
            this.captchaUrl = URL.createObjectURL(blob);
          } else {
            showDanger('Failed to load verification code, Click to refresh');
          }
        });
      }
    },

    // switch login method
    switchLoginType(type) {
      this.isMobileLogin = type === 'mobile';
      // Clear form
      this.form.username = '';
      this.form.mobile = '';
      this.form.password = '';
      this.form.captcha = '';
      this.fetchCaptcha();
    },

    // Form validation
    validateInput(input, message) {
      if (!input.trim()) {
        showDanger(message);
        return false;
      }
      return true;
    },

    async login() {
      if (this.isMobileLogin) {
        // mobile number validation
        if (!validateMobile(this.form.mobile, this.form.areaCode)) {
          showDanger('Please enter a valid mobile number');
          return;
        }
        // concate mobile number as username
        this.form.username = this.form.areaCode + this.form.mobile;
      } else {
        // username validation
        if (!this.validateInput(this.form.username, 'Username can not be empty')) {
          return;
        }
      }

      // password validation
      if (!this.validateInput(this.form.password, 'Password can not be empty')) {
        return;
      }
      // verification code validation
      if (!this.validateInput(this.form.captcha, 'Verification code can not be empty')) {
        return;
      }

      this.form.captchaId = this.captchaUuid
      Api.user.login(this.form, ({ data }) => {
        showSuccess('Login successfully!');
        this.$store.commit('setToken', JSON.stringify(data.data));
        goToPage('/home');
      }, (err) => {
        showDanger(err.data.msg || 'Login failed')
        if (err.data != null && err.data.msg != null && err.data.msg.indexOf('Verification code') > -1) {
          this.fetchCaptcha()
        }
      })

      // fetch verification code again
      setTimeout(() => {
        this.fetchCaptcha();
      }, 1000);
    },

    goToRegister() {
      goToPage('/register')
    },
    goToForgetPassword() {
      goToPage('/retrieve-password')
    },
  }
}
</script>
<style lang="scss" scoped>
@import './auth.scss';

.login-type-container {
  margin: 10px 20px;
}

:deep(.el-button--primary) {
  background-color: #5778ff;
  border-color: #5778ff;

  &:hover,
  &:focus {
    background-color: #4a6ae8;
    border-color: #4a6ae8;
  }

  &:active {
    background-color: #3d5cd6;
    border-color: #3d5cd6;
  }
}
</style>
