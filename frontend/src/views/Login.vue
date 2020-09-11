<template>
  <div id="login" :class="{ recaptcha: recaptcha }">
    <div class="main">
      <img :src="logoURL" alt="File Browser">
      <h1>{{ name }}</h1>
      <div class="card">
        <div class="auto">
          <h3>Connexion automatique</h3>
          <div class="items">
            <form @submit="submitAutologin" v-for="login in logins" :key="login.username">
              <input class="button button--block" type="submit" :value="login.username">
            </form>
          </div>
        </div>
        <div class="divider"></div>
        <form @submit="submit">
          <h3>Connexion manuelle</h3>
          <div v-if="error !== ''" class="wrong">{{ error }}</div>

          <input class="input input--block" type="text" v-model="username" :placeholder="$t('login.username')">
          <input class="input input--block" type="password" v-model="password" :placeholder="$t('login.password')">
          <input class="input input--block" v-if="createMode" type="password" v-model="passwordConfirm" :placeholder="$t('login.passwordConfirm')" />

          <div v-if="recaptcha" id="recaptcha"></div>
          <input class="button button--block" type="submit" :value="createMode ? $t('login.signup') : $t('login.submit')">

          <p @click="toggleMode" v-if="signup">{{ createMode ? $t('login.loginInstead') : $t('login.createAnAccount') }}</p>
        </form>
      </div>
    </div>
  </div>
</template>

<script>
import * as auth from '@/utils/auth'
import { getAllAutologin } from '@/api/users'
import { name, logoURL, recaptcha, recaptchaKey, signup } from '@/utils/constants'

export default {
  name: 'login',
  computed: {
    signup: () => signup,
    name: () => name,
    logoURL: () => logoURL
  },
  data: function () {    
    return {
      createMode: false,
      error: '',
      username: '',
      password: '',
      recaptcha: recaptcha,
      passwordConfirm: '',
      logins: []
    }
  },
  mounted () {
    this.resolveAutologin()

    if (!recaptcha) return

    window.grecaptcha.render('recaptcha', {
      sitekey: recaptchaKey
    })
  },
  methods: {
    resolveAutologin () {
      getAllAutologin().then((logins) => {
        this.logins = logins
      })
    },
    toggleMode () {
      this.createMode = !this.createMode
    },
    async submit (event) {
      event.preventDefault()
      event.stopPropagation()

      let redirect = this.$route.query.redirect
      if (redirect === '' || redirect === undefined || redirect === null) {
        redirect = '/files/'
      }

      let captcha = ''
      if (recaptcha) {
        captcha = window.grecaptcha.getResponse()

        if (captcha === '') {
          this.error = this.$t('login.wrongCredentials')
          return
        }
      }

      if (this.createMode) {
        if (this.password !== this.passwordConfirm) {
          this.error = this.$t('login.passwordsDontMatch')
          return
        }
      }

      try {
        if (this.createMode) {
          await auth.signup(this.username, this.password)
        }

        await auth.login(this.username, this.password, captcha)
        this.$router.push({ path: redirect })
      } catch (e) {
        if (e.message == 409) {
          this.error = this.$t('login.usernameTaken')
        } else {
          this.error = this.$t('login.wrongCredentials')
        }
      }
    },
    async submitAutologin (event) {
      event.preventDefault()
      event.stopPropagation()

      const username = event.target[0].value

      let redirect = this.$route.query.redirect
      if (redirect === '' || redirect === undefined || redirect === null) {
        redirect = '/files/'
      }

      try {
        await auth.login(username, "", "")
        this.$router.push({ path: redirect })
      } catch(e) {
        this.error = this.$t('login.wrongCredentials')
      }
    }
  }
}
</script>
